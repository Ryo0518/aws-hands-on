# Mission 11 - AWS基盤運用管理・可用性・監視・パッチ運用

## 1. ミッション概要

Mission 11では、社内Webシステムを想定し、AWS上に可用性と運用性を意識したWeb基盤を構築した。

主なテーマは以下のとおり。

- 2AZ構成による可用性確保
- ALBによる負荷分散
- Auto Scaling GroupによるEC2管理
- Private SubnetへのWebサーバー配置
- AWS Systems Managerによる運用管理
- VPC Endpointを利用したPrivate SubnetからのSSM接続
- CloudWatchによるCPU・メモリ・ステータス監視
- SNSによるアラーム通知
- Patch Managerによるパッチ適用とCompliance確認
- 障害・運用を想定した実機検証

Cognitoによる認証は、HTTPS・DNS・証明書・OAuth/OIDCまで含めると別ミッション相当のボリュームになるため、Mission 11では構築せず、Mission 12で扱うこととした。

---

## 2. 構築した構成

### 全体構成

```text
                         社内ユーザー
                              │
                              ▼
                    ┌─────────────────┐
                    │      ALB        │
                    │   2AZ構成       │
                    └────────┬────────┘
                             │
                    ┌────────┴────────┐
                    │                 │
                    ▼                 ▼
             ┌─────────────┐   ┌─────────────┐
             │    EC2-A    │   │    EC2-B    │
             │ Private Sub │   │ Private Sub │
             │   AZ-a      │   │   AZ-c      │
             └─────────────┘   └─────────────┘
                    │                 │
                    └────────┬────────┘
                             │
                    Systems Manager
                             │
             ┌───────────────┴───────────────┐
             │                               │
        Session Manager                  Run Command
             │                               │
             └───────────────┬───────────────┘
                             │
                      CloudWatch Agent
                             │
                     CloudWatch Metrics
                             │
                         Alarm
                             │
                           SNS
```

---

## 3. ネットワーク構成

### VPC

- 既存Mission 11用VPCを使用
- 2AZ構成
- EC2はPrivate Subnetへ配置

### ALB

- 2AZにまたがって配置
- Target GroupにEC2を2台登録
- Health Checkを設定
- ALB DNS名からWebページへのアクセスを確認

### EC2

- 2台構成
- Private Subnetに配置
- Auto Scaling Groupで管理
- Target Groupに2台を登録
- Webサーバーとしてnginxを利用

---

## 4. Auto Scaling Group

### 構成

- Minimum: 2
- Desired: 2
- Maximum: 2

今回は自動的な台数増減そのものよりも、

> 「2AZにWebサーバーを分散配置し、1台に問題があってもサービスを継続できる構成」

を学習目的とした。

### 確認内容

- EC2 2台がTarget Groupに登録されていること
- Target GroupのHealth CheckがHealthyになること
- ALB DNSからWebページが表示されること

---

## 5. AWS Systems Manager

Mission 11では、Private Subnetに配置したEC2を運用するため、Systems Managerを重点的に学習した。

### 使用した機能

- Session Manager
- Run Command
- Patch Manager

### IAM

EC2にSystems Manager利用に必要なIAMロールをアタッチ。

これにより、SSH用の踏み台サーバーを用意せず、Session ManagerからEC2へ接続できる構成とした。

---

## 6. VPC Endpoint

Private SubnetにあるEC2からSystems Managerを利用するため、VPC Endpointを構築した。

使用した主なエンドポイント：

- `ssm`
- `ssmmessages`
- `ec2messages`

EC2のSecurity GroupからHTTPS（443）でEndpointへ通信できるよう設定。

### 学習ポイント

Private SubnetのEC2では、単純にインターネットへ出られなくても、VPC Endpointを利用することでAWSサービスへPrivateに接続できる。

---

## 7. Session Manager

Session ManagerからPrivate SubnetのEC2へ接続できることを確認。

### 確認したこと

- EC2がSystems ManagerでManaged Nodeとして認識される
- Session Managerからシェルへ接続できる
- SSHポートを直接公開せずに運用できる

---

## 8. Run Command

Run Commandを利用して、複数のEC2へ一括でコマンドを実行した。

### 学習したこと

個々のEC2へログインして作業するのではなく、

```text
Systems Manager
       │
       ▼
Run Command
       │
   ┌───┴───┐
   ▼       ▼
 EC2-A   EC2-B
```

という形で複数インスタンスをまとめて管理できる。

これは、実際のAWS基盤運用で重要になる「大量のサーバーを効率的に管理する」という考え方につながる。

---

## 9. CloudWatch監視

Mission 11では、OSレベルのメトリクスをCloudWatchで監視するため、CloudWatch Agentを導入した。

### 監視したメトリクス

- CPU使用率
- メモリ使用率
- EC2 Status Check

### CloudWatch Agent

CloudWatch Agentから、

```text
mem_used_percent
```

をCloudWatchへ送信。

Private SubnetからCloudWatchへメトリクスを送信するため、CloudWatch Monitoring Endpointへの通信経路を確認した。

---

## 10. CloudWatch Alarm

### CPU監視

ASG全体のCPU使用率を監視するアラームを作成。

CPU負荷を意図的に発生させ、アラームが発火することを確認した。

### メモリ監視

CloudWatch Agentの

```text
mem_used_percent
```

を利用してメモリ使用率を監視。

しきい値を80%に設定し、負荷を発生させてアラームを発火させた。

その後、負荷を停止してメモリ使用率が低下し、アラームがOK状態へ戻ることも確認した。

### EC2 Status Check

2台のEC2それぞれに、

```text
StatusCheckFailed_Instance
```

のアラームを設定。

---

## 11. SNS通知

CloudWatch AlarmとSNSを連携。

アラーム状態になった際にメール通知を受け取れる構成を構築した。

流れ：

```text
CloudWatch Metric
       ↓
CloudWatch Alarm
       ↓
       SNS
       ↓
     Email
```

これにより、単にメトリクスを確認するだけではなく、障害・異常を検知した際に運用担当者へ通知する仕組みを構築した。

---

## 12. Patch Manager

Systems Manager Patch Managerを利用して、EC2のパッチ運用を実施した。

### 実施した流れ

```text
Patch Manager
      ↓
Scan
      ↓
Compliance確認
      ↓
Non-Compliant
      ↓
パッチ適用
      ↓
Pending Reboot
      ↓
再起動
      ↓
再Scan
      ↓
Compliant
```

最終的に、Mission 11の2台のEC2がともに **Compliant** となることを確認した。

### Pending Reboot

パッチ適用後に、

```text
InstalledPendingReboot
```

の状態が確認できた。

これはパッチ自体はインストールされたものの、再起動が必要な状態であることを確認する良い実運用上のポイントとなった。

---

## 13. 定期パッチ運用の検討

以下の月次運用を設計した。

- 適用頻度：毎月
- 適用日時：毎月1日の25時を想定
- 対象：Mission 11のTarget Groupに登録されたEC2
- 再起動：必要な場合は自動再起動
- 適用後確認：Patch ManagerのCompliance

Patch Policyのカスタムスケジュールについては、希望する日時をそのまま設定できない制約があり、Maintenance Windowまで使った実装は今回は行わないこととした。

今回のMissionでは、

> Patch ManagerによるScan → Install → Reboot → Compliance確認

という基本的な月次パッチ運用の流れを理解・実践できたことを成果とする。

---

## 14. トラブルシューティング

### CloudWatch AgentからCloudWatchへ送信できない問題

CloudWatch Agentログで以下のエラーを確認した。

```text
RequestError: send request failed
Post "https://monitoring.ap-northeast-1.amazonaws.com/":
dial tcp ...:443: i/o timeout
```

当初、CloudWatchへの通信経路に問題があることを疑った。

### DNS確認

```bash
nslookup monitoring.ap-northeast-1.amazonaws.com
```

により、CloudWatch Monitoring EndpointがVPC内のPrivate IPへ名前解決されることを確認。

### HTTPS疎通確認

```bash
curl -v --connect-timeout 5 \
https://monitoring.ap-northeast-1.amazonaws.com/
```

を実行。

結果として、

```text
SSL connection using TLSv1.3
```

となり、TLSハンドシェイクが成功。

さらに、

```text
HTTP/1.1 404 Not Found
<UnknownOperationException/>
```

が返った。

これは「CloudWatch APIへのGETリクエストとしては正しいAPI操作ではない」ものの、重要なのはHTTPS通信そのものが成立していることだった。

その後、CloudWatch Agentを再起動し、CloudWatch側でメトリクスが表示されることを確認した。

### 学習ポイント

障害調査では、

```text
DNS
 ↓
TCP/443
 ↓
TLS
 ↓
HTTP
 ↓
AWS API
```

のように通信を段階的に切り分けることが重要。

---

## 15. Cognito認証について

Mission 11ではCognitoによる認証も検討した。

想定構成：

```text
ユーザー
   ↓
ALB
   ↓
Cognito
   ↓
Google
   ↓
認証成功
   ↓
ALB
   ↓
EC2
```

Cognito User PoolやGoogle外部IdP、App Client、ALB ListenerのCognito認証などを設計段階まで検討した。

しかし、Googleログインまで実際に動かすためには、

- HTTPS
- ALB HTTPS Listener
- ACM
- DNS
- OAuth/OIDC
- Cognito
- Google IdP

など、別テーマの構築要素が多くなることが分かった。

そのため、Mission 11ではCognitoを実装せず、**Mission 12でHTTPS + 認証環境として独立して構築する**方針とした。

---

## 16. Mission 11で身についたこと

### AWS基盤

- VPC
- Subnet
- ALB
- Target Group
- EC2
- Auto Scaling Group

### 運用管理

- Systems Manager
- Session Manager
- Run Command
- Patch Manager
- Compliance

### 監視

- CloudWatch
- CloudWatch Agent
- CPU監視
- メモリ監視
- EC2 Status Check
- CloudWatch Alarm
- SNS通知

### ネットワーク

- Private Subnet
- VPC Endpoint
- AWSサービスへのPrivate通信
- DNS名前解決
- HTTPS疎通確認
- 通信障害の切り分け

### 運用設計

- 障害検知
- アラーム設計
- 通知
- パッチ適用
- 再起動
- Compliance確認

---

## 17. Mission 11の成果

Mission 11では、単にAWSサービスを触るだけではなく、

> 「構築したAWS環境を継続的に運用・監視する」

という視点で環境を構築した。

特に、

```text
構築
 ↓
監視
 ↓
異常検知
 ↓
通知
 ↓
運用対応
 ↓
パッチ適用
 ↓
Compliance確認
```

という一連の運用サイクルを実際に経験できた。

これは、AWS基盤運用管理チームで求められる、

- プロアクティブな監視
- 障害の早期検知
- 障害対応
- パッチ運用
- サーバー運用の効率化

につながる実践的な学習となった。

---

## 18. Mission 12への接続

Mission 12では、Mission 11で構築した基盤を発展させ、

**「HTTPS + 認証を備えたWeb基盤」**

をテーマとする。

候補構成：

```text
                    Google
                      │
                   OAuth/OIDC
                      │
                      ▼
                Cognito User Pool
                      │
                      ▼
                  ALB HTTPS
                      │
                 Cognito認証
                      │
                      ▼
                 Target Group
                  /        \
                 ▼          ▼
              EC2-A       EC2-B
```

学習予定：

- Route 53
- ACM
- HTTPS
- TLS証明書
- Cognito User Pool
- App Client
- Google外部IdP
- OAuth 2.0 / OIDC
- ALB Authenticate with Cognito
- 認証フロー
- JWT / Cookieの理解

Mission 11で構築したALB・EC2・Systems Manager・CloudWatch等の基盤を再利用し、Mission 12では「セキュアなWeb基盤」へ発展させる。
