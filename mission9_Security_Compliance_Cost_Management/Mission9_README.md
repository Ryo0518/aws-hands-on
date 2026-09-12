# AWS Mission 9 --- セキュリティ・コンプライアンス・コスト管理の自動化

## 1. ミッション概要

**テーマ：セキュリティ・コンプライアンス・コスト管理の自動化**

AWS環境への不正アクセス防御・ログ分析、コスト異常の検知・原因調査を通して、AWSインフラ運用に必要なセキュリティ／コスト管理を実践した。

### 主な使用サービス

-   AWS WAF
-   Amazon S3
-   AWS CLI
-   AWS Billing / Cost Management
-   Cost Explorer
-   Cost Anomaly Detection
-   NAT Gateway

> AWS Config、Amazon Inspector、EventBridge、Lambda、SNS、AWS
> Budgetsは当初の候補として整理したが、Mission
> 9では必須構築まで行わず、WAFログ分析とコスト管理を中心に完了とした。

------------------------------------------------------------------------

## 2. WAFによるアクセス防御

-   Web ACL：`mission9-web-ACL`
-   ルール：`mission9-IP-rule`
-   許可した自分のグローバルIPv4以外をBLOCKするIP制限を設定。
-   WAFログをS3へ保存。
-   S3バケット：`aws-waf-logs-mission9-blocked-ip-bucket`

構成：

``` text
Internet
   ↓
AWS WAF
   ↓
許可IP？
 ┌─ Yes ─→ ALLOW → EC2
 └─ No  ─→ BLOCK
```

------------------------------------------------------------------------

## 3. AWS CLIでWAFログを分析

9/9のログをS3から一括取得し、CLIとLinuxコマンドで分析した。

### ログファイル数

**137ファイル**

### WAFログレコード数

**10,656件**

### BLOCK件数

**10,656件**

今回確認した9/9のログレコードは全件BLOCKだった。

使用した主なコマンド：

``` bash
aws s3 cp s3://aws-waf-logs-mission9-blocked-ip-bucket/AWSLogs/141145165044/WAFLogs/ap-northeast-1/mission9-web-ACL/2026/09/09/ ~/waf-logs/2026-09-09/ --recursive --no-progress
```

``` bash
find ~/waf-logs/2026-09-09 -name "*.log.gz" -print0 | xargs -0 -n 1 gzip -dc | wc -l
```

``` bash
find ~/waf-logs/2026-09-09 -name "*.log.gz" -print0 | xargs -0 -n 1 gzip -dc | jq -r '.action' | sort | uniq -c | sort -nr
```

------------------------------------------------------------------------

## 4. 最大のアクセス元IP

`34.58.172.67` が **10,278件**を占め、9/9全体の約96.5%だった。

### 時間帯

-   19:00 UTC：3,545件
-   20:00 UTC：6,733件
-   合計：10,278件

日本時間では9/10 04:00〜06:00頃に集中。

### URI

10,278件に対して、**9,524種類のユニークURI**を確認。

代表例：

``` text
/index.php
/index.cgi
/index.jsp
/index.asp
/index.aspx
/login
/login.php
/admin/index.php
/struts/utils.js
/struts2-showcase/struts/utils.js
/cgi-bin/powerup/r.cgi
/EXCU_SHELL
/quixplorer_2_3/index.php
/phpmygallery/_conf/
```

PHP、JSP、ASP、CGI、Struts、CMS、管理画面などを横断して探索していた。

------------------------------------------------------------------------

## 5. User-Agent分析

`34.58.172.67` の主なUser-Agent：

-   10,105件：Chrome 133系を名乗るUser-Agent
-   4件：`Nmap Scripting Engine`
-   その他、古いFirefox / Edge / IE系など
-   JNDI系の文字列
-   `"; system(id);#`

確認されたJNDI系の例：

``` text
${jndi:ldap://...}
```

これらから、通常の人間による閲覧ではなく、**自動化されたWeb脆弱性スキャン／探索の可能性が非常に高い**と評価した。

ただし、User-Agentやペイロードだけから特定組織・ツールの実行者を断定しない。

------------------------------------------------------------------------

## 6. 国別分析

9/9のBLOCKログを国別に集計した。

  国コード       件数
  ---------- --------
  US           10,393
  IS              139
  TR               44
  BR               13
  BG               12
  NL               10
  IN                9
  ZA                6
  SE                4
  PT                3
  その他           23

`34.58.172.67` はAWS WAF上では **US** と判定された。

------------------------------------------------------------------------

## 7. 別のアクセス元 `157.157.221.26`

アイスランド（IS）の139件はすべて `157.157.221.26` からだった。

代表的な探索対象：

``` text
/.env/
/.ENV
/%252eenv
/web/.env
/storage/.env
/src/.env
/script/.env

/wp-config.php
/wp-config.php.bak
/wp-config.php.old
/wp-config.php.save
/wordpress/wp-config.php

/config.json
/appsettings.json
/secrets.yml
/secrets.yaml
/secrets.json

/www.git/
/www.git/HEAD
/www.git/config
```

`.env`、WordPress設定、各種secret、Gitリポジトリなど、**秘密情報やソースコードの露出を探す探索**と評価した。

------------------------------------------------------------------------

## 8. 総合的なセキュリティ評価

### `34.58.172.67`

-   10,278件
-   約2時間に集中
-   9,524種類のURI
-   多数のWeb技術・管理画面・CGI等を探索
-   Nmap系User-Agent
-   JNDI系ペイロード
-   コマンドインジェクションを疑わせる文字列

**評価：自動化されたWeb脆弱性スキャン／探索の可能性が非常に高い。**

### `157.157.221.26`

-   139件
-   `.env`
-   `.git`
-   `wp-config.php`
-   `secrets.*`
-   `config.json`
-   `appsettings.json`

**評価：設定ファイル・秘密情報・ソースコードの露出を探す自動探索の可能性が高い。**

### WAFを通していたら？

EC2ではnginxを中心に稼働していたため、スキャン対象の多くは存在しない可能性が高い。しかし、もし脆弱なアプリケーションや公開設定ファイルが存在していれば、情報漏洩・コード実行・認証情報窃取などにつながる可能性がある。

今回確認できたログでは、対象リクエストはWAFでBLOCKされており、**このログから侵入成功を示す事実は確認されていない**。

------------------------------------------------------------------------

## 9. コスト管理

IAMユーザーからBilling / Cost Managementへアクセスできることを確認。

確認した項目： - 月初来コスト - 前月との比較 - 当月予想コスト - Cost
Anomaly Detection - Cost Explorer

------------------------------------------------------------------------

## 10. Cost Anomaly Detection

2026/08/30のコスト異常を確認。

-   予想支出：**\$1.57**
-   実際の支出：**\$3.03**
-   コストへの影響：**\$1.46**
-   期間：1日

根本原因としてNAT Gateway関連の使用が表示された。

------------------------------------------------------------------------

## 11. Cost Explorerで原因調査

NAT Gatewayの使用量を日別に確認。

  日付             使用量       コスト
  ---------- ------------ ------------
  8/27             34時間       \$2.11
  8/28              8時間       \$0.50
  8/29              4時間       \$0.25
  8/30             48時間       \$2.98
  **合計**     **94時間**   **\$5.83**

8/30は48時間だったため、**NAT
Gatewayが2台、約24時間稼働していた可能性が高い**と推測した。

Cost Anomaly DetectionからCost
Explorerへ移り、異常の原因を使用量・コストまで掘り下げる流れを体験した。

------------------------------------------------------------------------

## 12. Mission 9で学んだこと

### AWS WAF

-   Web ACL
-   IP制限
-   BLOCK
-   WAFログ
-   S3へのログ保存
-   不審アクセスの分析

### AWS CLI

今回の大きな学習成果。

``` text
AWS CLI
  ↓
S3から一括取得
  ↓
gzip
  ↓
jq
  ↓
grep / sort / uniq / wc
  ↓
大量ログを効率的に分析
```

GUIで多数のログファイルを個別確認するのではなく、CLIで一括取得・集計・分析できることを実感した。

### コスト管理

-   Billing / Cost Management
-   Cost Explorer
-   Cost Anomaly Detection
-   NAT Gatewayのコスト
-   使用量とコストの関係

を実際のAWS環境で確認した。

------------------------------------------------------------------------

## 13. Mission 9 総括

Mission 9では、以下の運用フローを実際に経験した。

``` text
【セキュリティ】
外部アクセス
    ↓
AWS WAFで防御
    ↓
S3へログ保存
    ↓
AWS CLIで一括取得
    ↓
jq等で分析
    ↓
不審アクセスを評価
```

``` text
【コスト】
コスト異常
    ↓
Cost Anomaly Detection
    ↓
根本原因を確認
    ↓
Cost Explorer
    ↓
使用量・コストを分析
```

特に、**AWSコンソールで構築するだけでなく、CLIを使って実際のログを調査・運用できたこと**が大きな成果。

## Mission 9：COMPLETED

次のMissionでは、Mission 9で身につけたAWS
CLI・ログ分析・コスト分析の経験を活かし、さらに実務に近いAWSインフラ運用シナリオへ進む。
