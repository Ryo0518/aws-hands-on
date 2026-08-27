# Mission 7 --- イベント駆動型システム構築

## 1. 概要

Mission 7では、AWSのイベント駆動アーキテクチャを構築した。

CloudTrailで発生したAWSイベントをEventBridgeで検知し、SQSを経由してLambdaで処理する構成とした。

Lambdaでは受け取ったイベントを以下の3方向へ処理する。

-   S3：イベントJSONを保存
-   SNS：重要イベント（`IMPORTANT`）をメール通知
-   RDS：イベント情報をMySQLへ保存

また、EC2を踏み台としてSSM経由で接続し、RDSへの接続・読み書きも確認した。

------------------------------------------------------------------------

## 2. 構成

``` text
                    AWS CloudTrail
                         │
                         ▼
                  ┌─────────────┐
                  │ EventBridge │
                  └──────┬──────┘
                         │
                         ▼
                     ┌───────┐
                     │  SQS  │
                     └───┬───┘
                         │
                         ▼
                    ┌─────────┐
                    │ Lambda  │
                    └─┬──┬──┬─┘
                      │  │  │
              ┌───────┘  │  └────────┐
              ▼          ▼           ▼
             S3         SNS          RDS
          JSON保存    メール通知    MySQL保存
```

### ネットワーク

``` text
VPC
├── Public Subnet
│
├── Private Subnet 1
│   └── EC2
│
├── Private Subnet 2
│   └── Lambda
│
└── DB Subnet Group
    ├── Private Subnet 1
    └── Private Subnet 2
        └── RDS
```

EC2はSSM Session Managerから接続し、RDSはPrivate Subnetに配置した。

------------------------------------------------------------------------

## 3. 使用した主なAWSサービス

  サービス              役割
  --------------------- ----------------------------
  VPC                   ネットワーク基盤
  EC2                   RDS接続確認用・踏み台
  IAM                   AWSリソースへの権限付与
  SSM Session Manager   EC2への安全な接続
  RDS                   イベントデータのMySQL保存
  S3                    イベントJSONの保存
  Lambda                イベント処理
  Lambda Layer          PyMySQLライブラリの提供
  EventBridge           イベント検知・ルーティング
  SQS                   Lambdaへのイベント受け渡し
  SNS                   重要イベントのメール通知
  CloudWatch Logs       Lambda実行ログの確認
  CloudTrail            AWS API操作イベントの取得

------------------------------------------------------------------------

## 4. 構築内容

### VPC / ネットワーク

-   Mission 7用VPCを作成
-   Private Subnetを複数AZに配置
-   EC2をPrivate Subnetに配置
-   LambdaをPrivate Subnetに配置
-   RDS用DB Subnet Groupを作成

DB Subnet Groupには以下のPrivate Subnetを登録した。

-   `mission7-private-subnet1`
-   `mission7-private-subnet2`

複数AZにまたがる構成とした。

------------------------------------------------------------------------

## 5. EC2 / SSM

EC2を起動し、SSM Session Managerから接続できることを確認した。

EC2はRDSへの接続確認にも利用した。

### RDS接続時のトラブル

最初はEC2からRDSの3306番ポートへの接続がTimeoutした。

原因を切り分けた結果、EC2のSecurity
GroupのOutbound設定が誤っていたことが判明。

Outboundを修正したことで、

``` text
EC2 → TCP 3306 → RDS
```

の疎通に成功した。

その後、EC2からMySQLクライアントを使用してRDSへ接続できることを確認した。

------------------------------------------------------------------------

## 6. RDS

RDSをPrivate Subnetに配置。

DB Subnet Groupを作成し、複数AZのPrivate Subnetを登録した。

MySQLへ接続後、

``` sql
CREATE DATABASE mission7;
```

でデータベースを作成。

その後、イベント保存用のテーブルを作成した。

``` sql
CREATE TABLE events (
    id INT AUTO_INCREMENT PRIMARY KEY,
    event_type VARCHAR(50),
    message TEXT,
    source VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

手動INSERTを実行し、SELECTでデータが保存されることを確認した。

------------------------------------------------------------------------

## 7. Lambda

LambdaはSQSからイベントを受け取り、以下の処理を実行する。

1.  SQSメッセージからEventBridgeイベントを取得
2.  S3へイベントJSONを保存
3.  `eventType` が `IMPORTANT` の場合、SNSへ通知
4.  RDSの`events`テーブルへINSERT

### Lambda Layer

RDSのMySQLへ接続するため、Pythonライブラリの`PyMySQL`を使用。

Lambda標準環境にPyMySQLが含まれていないため、Lambda
Layerを作成して追加した。

Layer：

``` text
mission7-pymysql-layer
```

Lambdaコードから、

``` python
import pymysql
```

として利用できる状態にした。

### 環境変数

RDS接続情報はLambdaコードに直接記述せず、環境変数から取得する構成とした。

-   `DB_HOST`
-   `DB_PORT`
-   `DB_USER`
-   `DB_PASSWORD`
-   `DB_NAME`

------------------------------------------------------------------------

## 8. Security Group

RDSのSecurity Groupでは、以下からのMySQL通信を許可した。

``` text
Lambda Security Group
        │
        │ TCP 3306
        ▼
RDS Security Group
```

RDSをPublicに公開せず、Security
Groupによって接続元を制限する構成とした。

また、EC2からRDSへ接続確認する際には一時的にEC2のSecurity
Groupからの3306通信を許可した。

------------------------------------------------------------------------

## 9. EventBridge / SQS

EventBridgeでAWSイベントを検知し、SQSへイベントを送信。

SQSをLambdaのイベントソースとして設定した。

これにより、

``` text
EventBridge
    ↓
SQS
    ↓
Lambda
```

というイベント駆動型の処理フローを構築した。

------------------------------------------------------------------------

## 10. SNS

重要イベントをSNS TopicへPublish。

SNS Topic：

``` text
mission7-important-events
```

メールサブスクリプションを設定し、確認メールのリンクをクリックして`Confirmed`状態にした。

`IMPORTANT`イベントを発生させ、実際にメール通知されることを確認した。

------------------------------------------------------------------------

## 11. S3

イベントJSON保存用のS3バケットを作成。

LambdaからイベントをJSON形式で保存できることを確認した。

保存先：

``` text
events/YYYY/MM/DD/...
```

------------------------------------------------------------------------

## 12. 動作確認

### 手動テスト

SQSへテストイベントを送信。

Lambdaが正常に処理し、

-   S3にJSON保存
-   SNSからメール通知
-   RDSの`events`テーブルへINSERT

されることを確認した。

### 実イベントテスト

実際にEC2を起動・停止し、AWSイベントを発生させた。

その結果、

``` text
EC2 Start / Stop
      ↓
CloudTrail
      ↓
EventBridge
      ↓
SQS
      ↓
Lambda
 ┌────┼─────┐
 ▼    ▼     ▼
S3   SNS    RDS
```

という一連の処理が正常に動作した。

EC2の起動・停止イベントがRDSに記録されることも確認した。

------------------------------------------------------------------------

## 13. トラブルシューティング

### ① SNSメールが届かない

SNSのSubscriptionが`PendingConfirmation`だった。

確認メールが迷惑メールフォルダに入っていることを確認し、承認した。

その後、`Confirmed`になり、メール通知に成功。

### ② LambdaからSNS通知されない

Lambdaのテストイベント形式が実際のEventBridgeイベント形式と異なっていた。

テストイベントを実際のイベント構造に合わせることで解決。

### ③ EC2 → RDSがTimeout

EC2のSecurity GroupのOutbound設定に誤りがあった。

Outboundを修正後、

``` text
EC2 → RDS:3306
```

のTCP疎通に成功。

その後MySQL接続にも成功した。

### ④ LambdaからRDSへ接続するためのライブラリ

Lambda標準環境にPyMySQLが含まれていないため、Lambda Layerを作成。

PyMySQL Layerを追加後、LambdaからRDSへのINSERTに成功した。

------------------------------------------------------------------------

## 14. Mission 7で学んだこと

### イベント駆動アーキテクチャ

AWSサービス間をイベントで連携する基本的な構成を実践した。

``` text
EventBridge → SQS → Lambda
```

### SQS

イベントをLambdaへ直接渡すのではなく、SQSを間に入れることで、イベントをキューとして受け渡す構成を経験した。

### Lambda

Lambdaで、

-   S3
-   SNS
-   RDS

の複数サービスを操作する処理を実装した。

### Lambda Layer

Lambdaに標準搭載されていないPythonライブラリをLayerとして追加する方法を学習した。

### Security Group

Security
GroupのInboundだけでなく、Outboundも通信に影響することを実際のトラブルシューティングを通して理解した。

### RDS

Private
Subnetに配置したRDSへ、VPC内部のEC2やLambdaから接続する構成を実践した。

### JSON

EventBridgeやSQSで扱うイベントデータをJSONとして理解し、LambdaでJSONから必要な値を取り出す処理を経験した。

------------------------------------------------------------------------

## 15. 今後の学習課題

Mission 7を通して、以下を今後の課題とする。

-   JSONを自力で記述できるようになる
-   JSONのオブジェクト、配列、ネストを理解する
-   Pythonの基本構文を少しずつ身につける
-   Lambdaコードを読んで処理内容を説明できるようにする
-   既存Lambdaコードを自分で変更できるようにする
-   AWS CLIで今回のような構成を自力構築できるようにする

特にJSONについては、まず簡単なオブジェクトを自分で書けるレベルから練習する。

------------------------------------------------------------------------

## 16. Mission 7 完了

Mission
7では、AWSのイベント駆動型アーキテクチャを実際に構築し、以下の一連の処理を実証した。

``` text
AWSイベント
    ↓
CloudTrail
    ↓
EventBridge
    ↓
SQS
    ↓
Lambda
 ┌────┼─────┐
 ▼    ▼     ▼
S3   SNS    RDS
```

**Mission 7 完了。**
