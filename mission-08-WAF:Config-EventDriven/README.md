# Mission 8 — AWS Config × EventBridge × SQS × Lambda × S3 × SNS

## 1. 概要

AWS ConfigでAWSリソースの設定変更・コンプライアンス違反を検知し、EventBridgeを起点としてSQS、Lambdaへイベントを連携するイベント駆動型の監視・通知システムを構築した。

Lambdaでは、AWS ConfigのイベントをJSONとして解析し、

- S3へイベント情報をJSON形式で保存
- SNSへ通知をPublish
- SNSからメール通知

を行う構成とした。

---

## 2. 構成

```text
                         AWS Config
                             │
                             │ Config Rules
                             │ Compliance Change
                             ▼
                       EventBridge
                             │
                             │ Event Pattern
                             │ restricted-ssh
                             │ NON_COMPLIANT
                             ▼
                           SQS
                    mission8-queue
                             │
                             │ Lambda Trigger
                             ▼
                          Lambda
                             │
                    ┌────────┴────────┐
                    │                 │
                    ▼                 ▼
                   S3                SNS
             JSONイベント保存          │
                                      ▼
                                    Email
```

---

## 3. 使用したAWSサービス

| サービス | 役割 |
|---|---|
| AWS Config | AWSリソースの設定状態を評価 |
| EventBridge | Configのイベントを検知・フィルタリング |
| Amazon SQS | イベントをキューイング |
| AWS Lambda | SQSからイベントを取得して処理 |
| Amazon S3 | ConfigイベントをJSONとして保存 |
| Amazon SNS | 非準拠イベントをメール通知 |
| IAM | 各AWSサービス間のアクセス権限を管理 |
| CloudWatch Logs | Lambdaの実行ログを確認 |

---

## 4. AWS Config

Config Ruleとして `restricted-ssh` を使用。

EC2 Security GroupのSSHアクセスがインターネットに対して無制限になっていないかを評価する。

### 動作確認

Security Groupの設定を意図的に変更し、`restricted-ssh` を

```text
COMPLIANT
    ↓
NON_COMPLIANT
```

へ変化させた。

AWS Config上で対象Security Groupが非準拠として検出されることを確認した。

---

## 5. EventBridge

AWS Configの `Config Rules Compliance Change` イベントをトリガーとして使用。

イベントパターン：

```json
{
  "source": ["aws.config"],
  "detail-type": ["Config Rules Compliance Change"],
  "detail": {
    "configRuleName": ["restricted-ssh"],
    "newEvaluationResult": {
      "complianceType": ["NON_COMPLIANT"]
    }
  }
}
```

これにより、

> `restricted-ssh` が `NON_COMPLIANT` になった場合

だけ後続処理を実行する。

### ポイント

AWS ConfigからEventBridgeへのイベント連携では、今回の構成でEventBridgeにConfigの読み取り用IAMロールを個別に付与する必要はない。

---

## 6. SQS

キュー：

```text
mission8-queue
```

EventBridgeから送られたConfigイベントを一度SQSへ格納する。

```text
EventBridge
     ↓
    SQS
     ↓
  Lambda
```

SQSを利用することで、イベントを一時的に保持しながらLambdaで非同期処理できる。

---

## 7. Lambda

SQSをトリガーとしてLambdaを実行。

LambdaではSQSの `Records` からメッセージを取得し、`body` に格納されているJSON文字列を解析する。

```python
body = record["body"]

config_event = json.loads(body)
```

`json.loads()` によってJSON文字列をPythonの辞書として扱えるようにした。

---

## 8. S3への保存

LambdaからS3へConfigイベントをJSON形式で保存。

```text
mission8-putobject-bucket-20260828
└── config-events/
    └── <Event ID>.json
```

保存処理：

```python
s3.put_object(
    Bucket=BUCKET_NAME,
    Key=key,
    Body=json.dumps(config_event, indent=2),
    ContentType="application/json"
)
```

`json.dumps()` によってPythonのデータをJSON形式に戻して保存した。

実際にS3上でJSONファイルが生成され、Configイベントの内容を確認できることを確認した。

---

## 9. SNS通知

LambdaからSNSトピックへメッセージをPublish。

```python
sns.publish(
    TopicArn=SNS_TOPIC_ARN,
    Subject="AWS Config 非準拠検知",
    Message=message
)
```

SNSトピックにはEメールをSubscriptionとして設定。

最終的に、

```text
AWS Config
    ↓
EventBridge
    ↓
SQS
    ↓
Lambda
    ↓
SNS
    ↓
Email
```

という通知経路を構築した。

実際に非準拠状態を発生させ、SNSによるメール通知が届くことを確認した。

---

## 10. IAM権限

Lambda実行ロールに以下の権限を設定した。

### SQS

```text
sqs:ReceiveMessage
sqs:DeleteMessage
sqs:GetQueueAttributes
```

| アクション | 役割 |
|---|---|
| `sqs:ReceiveMessage` | SQSからメッセージを取得 |
| `sqs:DeleteMessage` | 処理済みメッセージを削除 |
| `sqs:GetQueueAttributes` | SQSキューの属性情報を取得 |

### CloudWatch Logs

```text
logs:CreateLogGroup
logs:CreateLogStream
logs:PutLogEvents
```

Lambdaの実行ログをCloudWatch Logsへ出力するために使用。

### S3

```text
s3:PutObject
```

LambdaからS3へJSONオブジェクトを書き込むために使用。

対象バケットに対する書き込みだけを許可するインラインポリシーを設定した。

### SNS

```text
sns:Publish
```

LambdaからSNSトピックへメッセージをPublishするために使用。

---

## 11. 動作確認

以下の一連の動作を実際に確認した。

### ① Config Ruleを非準拠に変更

Security Groupの設定を変更。

```text
restricted-ssh
COMPLIANT
    ↓
NON_COMPLIANT
```

### ② EventBridgeがイベントを検知

ConfigのCompliance ChangeイベントをEventBridgeが検知。

### ③ SQSへメッセージ送信

EventBridgeから `mission8-queue` へイベントが送信されることを確認。

### ④ LambdaがSQSイベントを取得

LambdaがSQSをトリガーとして実行されることを確認。

CloudWatch LogsでSQSイベントの内容を確認した。

### ⑤ S3へ保存

LambdaによってConfigイベントのJSONファイルが生成されることを確認。

S3上のJSONファイルを開き、Configイベントの内容を確認した。

### ⑥ SNS通知

LambdaからSNSへPublishされ、Subscriptionしているメールアドレスへ通知が届くことを確認。

---

## 12. 今回学んだこと

### イベント駆動アーキテクチャ

AWS Configの状態変化を起点として、複数のAWSサービスを連携できることを学習した。

```text
イベント発生
    ↓
イベント検知
    ↓
キューイング
    ↓
処理
    ↓
保存・通知
```

### JSONデータの扱い

SQSの `body` はJSON文字列として渡される。

```python
json.loads(body)
```

でPythonのデータへ変換し、

```python
json.dumps(config_event)
```

でJSON形式へ戻してS3へ保存した。

### IAM

AWSサービスをLambdaから操作するには、Lambda実行ロールに必要な権限を付与する必要がある。

今回、

```text
SQS → ReceiveMessage
S3  → PutObject
SNS → Publish
```

という形で、AWS API操作とIAMアクションの対応関係を確認した。

### SNSとSQSの役割

SQSは、

> メッセージを一時的に保持して、確実に処理する

ためのサービス。

SNSは、

> 1つのメッセージを複数の購読先へ配信する

ためのサービス。

という役割の違いを確認した。

---

## 13. 最終成果

AWS Configのコンプライアンス違反を自動検知し、

```text
AWS Config
     ↓
EventBridge
     ↓
SQS
     ↓
Lambda
   ↙     ↘
 S3      SNS
          ↓
        Email
```

という実際に動作するイベント駆動型の監視・記録・通知システムを構築した。

単純なAWSサービス単体のハンズオンではなく、複数サービスをIAMとイベントで連携させる構成を実際に構築・検証できた。
