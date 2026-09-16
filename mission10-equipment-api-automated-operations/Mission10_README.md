# Mission 10：AWS Equipment Management API & Automated Operations

## ストーリー

社内で利用しているPCやその他のIT機器を管理するための「設備情報管理API」をAWS上に構築する。

単純なCRUD APIだけではなく、実運用を想定し、

- APIへの不正アクセス対策
- AWSリソースの設定変更検知
- AWS API操作の監査
- 障害発生時の検知・通知
- Slackへの自動通知
- Secrets Managerによる認証情報管理
- CloudTrailによる監査ログ保存
- コストを意識したAWSサービス構成

までを含めた、サーバーレスな運用基盤を構築する。

### 想定する利用シーン

社内ユーザーが設備情報をAPI経由で登録・参照・更新・削除する。

一方、AWS環境では、

- WAFの設定変更
- AWS API操作
- Lambdaの障害
- AWS Configによる非準拠検知

などを自動的に検知し、Slackへ通知する。

---

## 要件

### 1. システム構成

以下のAWSサービスを利用する。

- Amazon API Gateway
- AWS Lambda
- Amazon DynamoDB
- AWS WAF
- AWS Config
- Amazon EventBridge
- Amazon CloudWatch
- AWS CloudTrail
- Amazon S3
- AWS Secrets Manager
- AWS IAM

外部通知先としてSlackを利用する。

### 構成図

```text
                         Internet
                            │
                            ▼
                         AWS WAF
                            │
                            ▼
                  API Gateway REST API
                            │
                            ▼
                  Lambda: Equipment API
                            │
                            ▼
                       DynamoDB
                  mission10-equipment


        ┌──────────────── AWS Operations ────────────────┐
        │                                                 │
        │  AWS Config                                     │
        │      │                                          │
        │      ▼                                          │
        │  EventBridge                                    │
        │      │                                          │
        │      └──────────────┐                           │
        │                     ▼                           │
        │              Slack Notification Lambda          │
        │                     │                           │
        │                     ▼                           │
        │                   Slack                         │
        │                                                 │
        │  CloudWatch Alarm                               │
        │      │                                          │
        │      ▼                                          │
        │  EventBridge ───────────────► Slack              │
        │                                                 │
        │  CloudTrail                                     │
        │      │                                          │
        │      ├──────────────► S3                        │
        │      │              Audit Logs                  │
        │      │                                          │
        │      ▼                                          │
        │  EventBridge                                    │
        │      │                                          │
        │      └──────────────────────► Slack              │
        │                                                 │
        └─────────────────────────────────────────────────┘

              Secrets Manager
                    │
                    ▼
          Slack Bot Token
                    │
                    ▼
          Slack Notification Lambda
```

### 2. Equipment API

API Gateway REST APIをAPIエンドポイントとして使用する。

API Gateway：

```text
mission10-equipment-rest-api
```

Stage：

```text
prod
```

#### POST

```text
POST /equipment
```

設備情報を登録する。

例：

```json
{
  "equipmentId": "EQ001",
  "name": "MacBook Air M4",
  "type": "PC",
  "location": "Sapporo",
  "status": "available"
}
```

#### GET

```text
GET /equipment/{equipmentId}
```

指定した設備情報を取得する。

#### PUT

```text
PUT /equipment/{equipmentId}
```

指定した設備情報を更新する。

#### DELETE

```text
DELETE /equipment/{equipmentId}
```

指定した設備情報を削除する。

### 3. DynamoDB

テーブル名：

```text
mission10-equipment
```

パーティションキー：

```text
equipmentId
```

型：

```text
String
```

キャパシティモード：

```text
On-demand
```

今回は設備IDによる単一アイテムの取得を中心とするため、`Scan` や `Query` はLambdaから使用しない。

### 4. Lambda

#### Equipment API

関数名：

```text
mission10-equipment-api
```

役割：

- POST
- GET
- PUT
- DELETE

のAPI処理を担当する。

環境変数：

```text
TABLE_NAME=mission10-equipment
```

LambdaはVPCへ配置しない。

#### Slack Notification

関数名：

```text
mission10-event-notify-slack
```

役割：

EventBridgeから受け取ったイベントを解析し、Slackへ通知する。

対応イベント：

- AWS Config Compliance Change
- CloudWatch Alarm State Change
- AWS API Call via CloudTrail

### 5. API Gateway

REST APIを利用する。

理由：

AWS WAFによるWeb ACLをAPI Gatewayに関連付け、APIへのアクセス制御を行うため。

リソース：

```text
/equipment
/equipment/{equipmentId}
```

メソッド：

```text
POST
GET
PUT
DELETE
```

Lambda Proxy Integrationを使用する。

### 6. AWS WAF

Web ACL：

```text
mission10-waf
```

以下のルールを設定する。

#### IP制限

```text
mission10-IP-Block
```

許可対象のIPアドレス以外をBlockする。

#### AWS Managed Rules

```text
AWS-AWSManagedRulesCommonRuleSet
```

AWS Managed Rulesを利用して一般的なWeb攻撃への防御を行う。

#### Rate Limit

```text
Rate-Limit
```

5分間に100リクエストを超えたIPをBlockする。

Web ACLをAPI Gateway REST APIに関連付ける。

### 7. AWS Config

WAF Web ACLの設定状態を継続的に評価する。

Config Rule：

```text
mission10-waf-config-compliance
```

カスタムLambda：

```text
mission10-config-waf-check
```

期待するWAFルール：

```text
mission10-IP-Block
AWS-AWSManagedRulesCommonRuleSet
Rate-Limit
```

ルール構成が期待値と異なる場合、`NON_COMPLIANT` として検知する。

#### テスト

`Rate-Limit`を一時的に削除すると、`NON_COMPLIANT`となることを確認。

その後、`Rate-Limit`として復元すると、`COMPLIANT`に戻ることを確認した。

### 8. EventBridge

EventBridgeをイベント駆動処理の中心として使用する。

#### Config Compliance Change

対象：

```text
mission10-waf-config-compliance
```

`NON_COMPLIANT`を検知した場合、

```text
EventBridge
    ↓
mission10-event-notify-slack
    ↓
Slack
```

として通知する。

#### CloudWatch Alarm State Change

Alarm：

```text
mission10-lambda-errors
```

Lambdaのエラーを検知し、`ALARM`へ遷移した場合にSlack通知する。

#### CloudTrail API Call

WAFv2のAPI操作をCloudTrail経由で検知する。

対象：

```text
UpdateWebACL
DeleteWebACL
AssociateWebACL
DisassociateWebACL
```

イベントパターン：

```json
{
  "source": [
    "aws.wafv2"
  ],
  "detail-type": [
    "AWS API Call via CloudTrail"
  ],
  "detail": {
    "eventSource": [
      "wafv2.amazonaws.com"
    ],
    "eventName": [
      "UpdateWebACL",
      "DeleteWebACL",
      "AssociateWebACL",
      "DisassociateWebACL"
    ]
  }
}
```

### 9. CloudWatch

Equipment API Lambdaのエラーを監視する。

Alarm：

```text
mission10-lambda-errors
```

条件：

```text
Errors >= 1
```

評価期間：

```text
1分
```

1/1データポイントでALARMとする。

欠落データは「しきい値を超えていない」として扱う。

#### 障害試験

Lambdaに意図的な例外を発生させ、

```text
Lambda Error
    ↓
CloudWatch Alarm
    ↓
ALARM
    ↓
EventBridge
    ↓
Slack
```

の流れを確認した。

その後、Lambdaを正常なCRUDコードへ復旧し、Alarmが`OK`へ戻ることを確認した。

### 10. CloudTrail

Trail：

```text
mission10-cloudtrail
```

AWS API操作を監査ログとして記録する。

ログファイルはS3へ保存する。

CloudTrailとEventBridgeを組み合わせ、

```text
API操作
    ↓
CloudTrail
    ↓
EventBridge
    ↓
Slack
```

として重要なWAF操作を通知する。

#### テスト

WAFのルール順序を一時的に変更し、`UpdateWebACL`を発生させた。

その結果、

```text
CloudTrail
    ↓
EventBridge
    ↓
mission10-event-notify-slack
    ↓
Slack
```

で通知されることを確認した。

テスト後、WAFのルール順序を元に戻した。

### 11. Secrets Manager

Secret：

```text
mission10/slack-bot-token
```

Slack Bot Tokenを保存する。

Secret Key：

```text
SLACK_BOT_TOKEN
```

Slack TokenをLambdaコードへ直接記述せず、Secrets Managerから取得する。

Lambda IAMロールには、`secretsmanager:GetSecretValue` を許可する。

### 12. IAM

各Lambdaに専用IAMロールを作成し、必要最小限の権限を付与する。

Equipment API Lambda：

```text
dynamodb:PutItem
dynamodb:GetItem
dynamodb:UpdateItem
dynamodb:DeleteItem
```

Config Lambda：

```text
wafv2:GetWebACL
config:PutEvaluations
```

Slack通知Lambda：

```text
secretsmanager:GetSecretValue
```

用途ごとに権限を分離する。

### 13. セキュリティ

以下を実施する。

- WAFによるAPI保護
- IP制限
- AWS Managed Rules
- Rate Limit
- IAM最小権限
- Secrets ManagerによるToken管理
- AWS Configによる構成チェック
- CloudTrailによる監査
- S3への監査ログ保存
- EventBridgeによる変更検知
- Slackへの自動通知

### 14. 監視・障害対応

想定する障害フロー：

```text
障害発生
   ↓
CloudWatch
   ↓
Alarm
   ↓
EventBridge
   ↓
Slack
   ↓
担当者が状況確認
```

AWS Configによる非準拠：

```text
設定変更
   ↓
AWS Config
   ↓
NON_COMPLIANT
   ↓
EventBridge
   ↓
Slack
```

AWS API操作：

```text
AWS API操作
   ↓
CloudTrail
   ↓
EventBridge
   ↓
Slack
```

### 15. コスト管理

Mission 10では、以下のAWSサービスを利用する。

- API Gateway
- Lambda
- DynamoDB
- WAF
- Config
- EventBridge
- CloudWatch
- CloudTrail
- S3
- Secrets Manager

使用量を意識し、不要なリソースはMission終了後に削除する。

特に以下は継続利用によるコスト発生に注意する。

- AWS WAF
- AWS Config
- CloudTrail
- Secrets Manager
- CloudWatch Logs
- S3

### 16. 動作確認結果

#### CRUD

- [x] POSTで設備登録
- [x] GETで設備取得
- [x] PUTで設備更新
- [x] DELETEで設備削除
- [x] 削除後にGETすると404

#### WAF

- [x] IP制限
- [x] AWS Managed Rules
- [x] Rate Limit
- [x] API Gatewayとの関連付け

#### AWS Config

- [x] WAF設定をCOMPLIANTとして検知
- [x] Rate-Limit削除でNON_COMPLIANT
- [x] Rate-Limit復元でCOMPLIANT
- [x] NON_COMPLIANT時のSlack通知

#### CloudWatch

- [x] Lambdaエラーを検知
- [x] AlarmがALARMへ遷移
- [x] EventBridgeが検知
- [x] Slack通知
- [x] Lambda復旧後にAlarmがOKへ復帰

#### CloudTrail

- [x] CloudTrail Trail作成
- [x] S3への監査ログ保存
- [x] WAF API操作を検知
- [x] EventBridgeで検知
- [x] Slack通知

#### Secrets Manager

- [x] Slack Bot TokenをSecretへ保存
- [x] LambdaからSecret取得
- [x] Slackへの通知成功

### 17. Mission 10で学んだこと

- API Gateway REST APIとLambdaによるサーバーレスAPI構築
- DynamoDB CRUD
- API GatewayとWAFの連携
- WAFによるIP制限・Managed Rules・Rate Limit
- AWS Configによるリソース設定のコンプライアンス評価
- EventBridgeによるイベント駆動処理
- CloudWatch Alarmによる障害検知
- CloudTrailによるAWS API操作の監査
- CloudTrailとEventBridgeの連携
- Secrets Managerによる機密情報管理
- IAM最小権限
- Slack APIとの連携
- 障害発生から通知までの自動化
- AWS環境の変更検知と運用自動化

### 18. Mission 10の成果

Mission 10では、単純なAWSリソース構築だけではなく、

**「AWS環境を安全に運用し、異常や設定変更を検知して、必要な情報を自動的に通知する」**

ところまでを実際に構築した。

特に、

```text
監視
  ↓
検知
  ↓
イベント駆動
  ↓
自動通知
```

という運用設計を実環境で経験した。

また、WAF・Config・CloudTrail・CloudWatch・EventBridge・Secrets Managerを組み合わせることで、

**セキュリティ・コンプライアンス・監視・監査・通知**

を一つのシステムとして構成する経験を得た。
