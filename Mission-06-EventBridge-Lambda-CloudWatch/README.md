# Mission 6 --- EventBridge + Lambda + CloudWatch Logs

## 概要

EC2の状態変更イベントをAmazon
EventBridgeで検知し、対象イベントが発生したらAWS
Lambdaを呼び出して処理する構成を構築した。

Lambdaの実行結果はCloudWatch Logsで確認できるようにした。

今回のMission 6では、特に以下の流れを実際に構築・検証した。

``` text
EC2
  │
  │ Instance State-change Notification
  ▼
Amazon EventBridge
  │
  │ イベントパターン一致
  ▼
AWS Lambda
  │
  │ print()
  ▼
Amazon CloudWatch Logs
```

------------------------------------------------------------------------

## 構築したAWSサービス

-   Amazon EC2
-   Amazon EventBridge
-   AWS Lambda
-   Amazon CloudWatch Logs
-   AWS IAM
-   AWS CloudTrail
-   IAM Policy Simulator

------------------------------------------------------------------------

## 構成

### EventBridge

EC2の状態変更イベントを検知するEventBridgeルールを作成。

-   イベントバス：`default`
-   ルール名：`mission6-ec2-stop-rule`
-   イベントソース：Amazon EC2
-   イベントタイプ：EC2 Instance State-change Notification
-   ターゲット：
    -   Lambda
    -   CloudWatch Logs

EC2の停止イベントをトリガーとしてLambdaを実行する。

------------------------------------------------------------------------

## Lambda

### 関数

``` text
mission6-ec2-stopLogs
```

### コード

``` python
import json

def lambda_handler(event, context):
    print("=== Mission 6 Event Detected ===")
    print(json.dumps(event, indent=2, ensure_ascii=False))

    return {
        "statusCode": 200,
        "body": "EC2 stop event detected"
    }
```

### 処理内容

EventBridgeから渡されたイベントを受け取り、

1.  Mission 6のイベント検知メッセージを出力
2.  EventBridgeから渡されたイベントJSONをCloudWatch Logsへ出力
3.  HTTP 200相当のレスポンスを返す

という処理を行う。

------------------------------------------------------------------------

## IAM

今回のトラブルシュートでは、EventBridgeからLambdaを呼び出すための実行ロールを作り直した。

### EventBridge実行ロール

``` text
mission6-lambda-invoke-role
```

### 信頼ポリシー

EventBridgeがこのロールを引き受けられるようにする。

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "events.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### 許可ポリシー

EventBridgeが対象Lambdaを呼び出せるようにする。

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:ap-northeast-1:141145165044:function:mission6-ec2-stopLogs"
    }
  ]
}
```

今回はMission 6専用のインラインポリシーとして設定した。

------------------------------------------------------------------------

## CloudWatch Logs

Lambdaのロググループ：

``` text
/aws/lambda/mission6-ec2-stopLogs
```

LambdaがEventBridgeから呼び出されると、ログストリームが生成され、

``` text
=== Mission 6 Event Detected ===
```

およびEventBridgeから渡されたイベントJSONを確認できる。

------------------------------------------------------------------------

# 動作確認

## 1. Lambda単体テスト

まずLambdaコンソールからテストイベントを使用してLambda単体で実行。

正常に実行できることを確認した。

------------------------------------------------------------------------

## 2. EC2停止イベントの発生

対象EC2を停止。

EC2の状態変更イベントがEventBridgeに送信される。

------------------------------------------------------------------------

## 3. EventBridgeモニタリング

EventBridgeルールのモニタリングで以下のメトリクスを確認。

  メトリクス          意味
  ------------------- --------------------------------------
  MatchedEvents       イベントパターンに一致したイベント数
  Invocations         ターゲットを呼び出そうとした回数
  FailedInvocations   ターゲット呼び出しに失敗した回数
  TriggeredRules      ルールがトリガーされた回数

最終的な動作確認では、EventBridgeからLambdaへの呼び出しが正常に成功することを確認した。

------------------------------------------------------------------------

# トラブルシューティング

今回のMission
6では、EventBridgeのイベント検知自体は成功しているものの、Lambda側にログが出ない問題が発生した。

## 1. MatchedEventsを確認

最初にEventBridgeのモニタリングを確認。

``` text
MatchedEvents = 1
```

となっていたため、

> EC2のイベント → EventBridgeのイベントパターン

までは正常に動作していると判断した。

------------------------------------------------------------------------

## 2. Invocationsを確認

``` text
Invocations = 1～2
```

となっていたため、

> EventBridge → ターゲット呼び出し

まで進んでいることを確認した。

しかし、

``` text
FailedInvocations > 0
```

となるケースがあり、Lambda実行部分に問題があると判断した。

------------------------------------------------------------------------

## 3. CloudWatch Logsを確認

Lambdaのロググループが作成されていない／新しいログストリームが生成されない状態を確認。

そのためLambdaが正常に呼び出されていない可能性を調査した。

------------------------------------------------------------------------

## 4. CloudTrailでAssumeRoleを確認

AWS CloudTrailで、

``` text
eventSource = sts.amazonaws.com
eventName   = AssumeRole
```

を検索。

EventBridgeからIAMロールのAssumeRoleが成功していることを確認した。

CloudTrailイベントでは、

``` text
userIdentity.invokedBy = events.amazonaws.com
```

となっており、EventBridgeがロールを引き受けていることを確認できた。

------------------------------------------------------------------------

## 5. IAM Policy Simulatorで確認

IAM Policy Simulatorを使用して、

``` text
lambda:InvokeFunction
```

を対象Lambdaに対してシミュレーション。

結果が、

``` text
許可
```

となることを確認した。

つまり、

``` text
IAMロール
  ↓
lambda:InvokeFunction
  ↓
mission6-ec2-stopLogs
```

の権限自体は許可されていることを確認できた。

------------------------------------------------------------------------

## 6. EventBridge用実行ロールを作り直した

EventBridgeコンソールのロール作成ウィザードでは、信頼ポリシーのテンプレート変数

``` text
accountId
region
ruleName
```

を入力する必要があり、入力しても「テンプレート変数が必要」というエラーが残った。

そのため、EventBridgeのウィザードを使用する方法をやめ、IAMから通常のロールを作成した。

### 新しいロール

``` text
mission6-lambda-invoke-role
```

### 信頼関係

``` text
events.amazonaws.com
```

### 許可

``` text
lambda:InvokeFunction
```

対象：

``` text
mission6-ec2-stopLogs
```

その後、EventBridgeのLambdaターゲットにこのロールを紐付けた。

------------------------------------------------------------------------

# 最終確認

最終的に、

``` text
EC2停止
  ↓
EventBridge
  ↓
MatchedEvents
  ↓
Invocations
  ↓
mission6-lambda-invoke-role
  ↓
Lambda InvokeFunction
  ↓
mission6-ec2-stopLogs
  ↓
CloudWatch Logs
```

という一連の処理が正常に動作することを確認した。

------------------------------------------------------------------------

# 今回学んだこと

## EventBridgeは「イベントを検知するだけ」ではない

EventBridgeはイベントを検知したあと、ターゲットとなるAWSサービスを呼び出す。

今回の場合、

``` text
EC2
↓
EventBridge
↓
Lambda
```

というイベント駆動型の処理を構築した。

------------------------------------------------------------------------

## IAMロールには2種類の考え方がある

今回特に重要だった。

### 信頼ポリシー

「誰がこのロールを使えるか」

今回：

``` text
events.amazonaws.com
```

### 許可ポリシー

「このロールを使って何ができるか」

今回：

``` text
lambda:InvokeFunction
```

つまり、

``` text
信頼ポリシー
→ EventBridgeがロールを引き受けられる

許可ポリシー
→ そのロールでLambdaを呼び出せる
```

という関係。

------------------------------------------------------------------------

## CloudWatchのメトリクスから障害箇所を切り分けられる

今回のような問題では、

``` text
MatchedEvents
    ↓
Invocations
    ↓
FailedInvocations
    ↓
Lambda Logs
```

という順番で確認すると、どこまで処理が進んでいるか判断できる。

------------------------------------------------------------------------

## CloudTrailは「裏側で何が起きたか」を調査するのに使える

EventBridgeがIAMロールを引き受けたかどうかを、

``` text
CloudTrail
→ AssumeRole
```

から確認できた。

単に「Lambdaが動かない」と考えるのではなく、

``` text
イベント発生？
↓
EventBridge一致？
↓
Lambda呼び出し？
↓
IAM AssumeRole？
↓
Lambda権限？
↓
Lambda実行？
↓
CloudWatch Logs？
```

と分解して調査することが重要だと分かった。

------------------------------------------------------------------------

# Mission 6 完了

**EventBridgeによるEC2イベント検知 → Lambda実行 → CloudWatch
Logsへの出力までの一連のイベント駆動処理を構築・動作確認完了。**
