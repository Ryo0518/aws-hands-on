# Mission 5 --- サーバーレス＋監視を取り入れたWebシステム構築

## 1. 概要

Mission 5では、Mission 4の高可用性Webシステムをベースに、Auto
Scaling、CloudWatch、SNS、S3、Lambdaなどを追加し、**監視とサーバーレスを組み合わせたAWSシステム**を構築した。

AWSマネジメントコンソール（GUI）を中心に構築し、最後には実際にCPU負荷を発生させ、**CloudWatch
Alarm → SNS → メール通知**まで動作確認した。

------------------------------------------------------------------------

## 2. 目的

-   ALB + Auto Scalingによる可用性・スケーリングを理解する
-   NAT Gatewayを使ったPrivate Subnetからの外向き通信を理解する
-   CloudWatchによるメトリクス監視を理解する
-   CloudWatch AlarmとSNSを連携させる
-   S3イベントをトリガーにLambdaを実行する
-   Lambdaの実行ログをCloudWatch Logsで確認する
-   Systems Manager Session ManagerでPrivate Subnet内EC2へ接続する
-   複数のAWSサービスを組み合わせて一つのシステムとして動かす

------------------------------------------------------------------------

## 3. 最終アーキテクチャ

``` text
                         User
                           |
                        Internet
                           |
                    Internet Gateway
                           |
              +------------+------------+
              |           VPC            |
              |                          |
       Public Subnet 1            Public Subnet 2
       10.0.1.0/24                10.0.2.0/24
              |                          |
        NAT Gateway 1             NAT Gateway 2
              |                          |
              +---------+----------------+
                        |
                       ALB
                        |
               +--------+--------+
               |                 |
        Private Subnet 1  Private Subnet 2
         10.0.11.0/24      10.0.12.0/24
               |                 |
              EC2               EC2
               +--------+--------+
                        |
                       RDS
                     MySQL


        S3
         |
         | ObjectCreated
         v
      Lambda
         |
         v
  CloudWatch Logs


   EC2 / ASG
       |
       v
  CloudWatch
       |
       v
     Alarm
       |
       v
      SNS
       |
       v
     Email
```

API GatewayはMission 5には含めず、Mission 6へ持ち越す。

------------------------------------------------------------------------

## 4. ネットワーク構成

### VPC

-   Name: `mission5-vpc`
-   CIDR: `10.0.0.0/16`

### Public Subnet

  Subnet                    CIDR          AZ
  ------------------------- ------------- -----------------
  mission5-public-subnet1   10.0.1.0/24   ap-northeast-1a
  mission5-public-subnet2   10.0.2.0/24   ap-northeast-1c

### Private Subnet

  Subnet                     CIDR           AZ
  -------------------------- -------------- -----------------
  mission5-private-subnet1   10.0.11.0/24   ap-northeast-1a
  mission5-private-subnet2   10.0.12.0/24   ap-northeast-1c

### NAT Gateway

NAT Gatewayを2つ作成し、Public Subnetに1つずつ配置した。

Private Subnetからインターネットへ出る通信は、

``` text
Private EC2
 ↓
Private Route Table
 ↓
NAT Gateway
 ↓
Public Route Table
 ↓
Internet Gateway
 ↓
Internet
```

という経路になる。

------------------------------------------------------------------------

## 5. Route Table

### Public側

``` text
0.0.0.0/0 → Internet Gateway
```

Public Subnetに関連付けた。

### Private側

Private SubnetごとにNAT Gatewayへ向けたルートテーブルを作成。

``` text
0.0.0.0/0 → NAT Gateway
```

Private EC2から外部へ出る通信をNAT Gateway経由にした。

------------------------------------------------------------------------

## 6. Security Group

役割ごとにSecurity Groupを分離した。

### ALB用

-   HTTP 80
-   HTTPS 443
-   Webサーバー用Security Groupへの通信

### EC2用

-   HTTP 80
-   HTTPS 443
-   ALB用Security Groupからの通信

### RDS用

-   MySQL 3306
-   EC2用Security Groupからの通信

------------------------------------------------------------------------

## 7. RDS

MySQLのRDSを構築。

DB Subnet Group：

`mission5-rds-subnetgroup`

Private
Subnetを利用し、RDSをインターネットから直接アクセスできない構成とした。

------------------------------------------------------------------------

## 8. ベースEC2

Auto Scalingで利用するAMIのベースとなるEC2を作成。

ユーザーデータでNginxをインストールした。

``` bash
#!/bin/bash
dnf update -y
dnf -y install nginx
systemctl enable nginx
systemctl start nginx
```

NginxがActiveになっていることを確認した。

------------------------------------------------------------------------

## 9. 起動テンプレート

ASGから起動するEC2の設定を起動テンプレートにまとめた。

主な設定：

-   AMI
-   インスタンスタイプ
-   EC2用Security Group
-   Private Subnetで利用するネットワーク設定
-   SSM用IAMロール

------------------------------------------------------------------------

## 10. Application Load Balancer

ALBをInternet-facingで構築。

-   Name: `mission5-ALB`
-   Public Subnet 1
-   Public Subnet 2
-   HTTP : 80

ALBからTarget GroupへHTTP 80番で転送する。

------------------------------------------------------------------------

## 11. Target Group

Target Group：

`mission5-target-group`

設定：

-   Target type: Instances
-   Protocol: HTTP
-   Port: 80
-   Health Check Protocol: HTTP
-   Health Check Path: `/`
-   Success Code: 200

Auto Scaling Groupと連携した。

------------------------------------------------------------------------

## 12. Auto Scaling Group

Auto Scaling Group：

`mission5-auto-scaling-group`

### キャパシティ

-   Desired: 2
-   Minimum: 2
-   Maximum: 4

### スケーリング

Target Tracking Policyを使用。

-   Target CPU Utilization: 50%

ASGから2台のEC2が起動し、ALB経由でWebアクセスできることを確認した。

------------------------------------------------------------------------

## 13. CloudWatch

CloudWatchでEC2およびASGを監視。

主に以下のメトリクスを確認した。

-   CPUUtilization
-   NetworkIn
-   NetworkOut
-   NetworkPacketsIn
-   NetworkPacketsOut
-   ASGのインスタンス数関連メトリクス

ASGのメトリクス収集も有効化した。

------------------------------------------------------------------------

## 14. CloudWatch Alarm

Alarm名：

`mission5-high-cpu-alarm`

### 条件

-   Metric: CPUUtilization
-   Statistic: Average
-   Period: 5 minutes
-   Threshold: 70%
-   Condition: 70%以上

------------------------------------------------------------------------

## 15. SNSによるメール通知

SNS Topic：

`mission5-cloudwatch-alert`

CloudWatch AlarmからSNS Topicへ通知する構成を作成。

メールサブスクリプションを作成し、Confirm
subscriptionを実行して購読を確認した。

``` text
CloudWatch Alarm
       ↓
      SNS
       ↓
    Email
```

------------------------------------------------------------------------

## 16. 実負荷テスト

Systems Manager Session ManagerからEC2へ接続し、以下を実行。

``` bash
yes > /dev/null &
```

CPU使用率が上昇することをCloudWatchで確認した。

その後、

``` text
CPUUtilization
      ↓
70%以上
      ↓
CloudWatch Alarm
      ↓
ALARM
      ↓
SNS
      ↓
メール通知
```

という一連の動作を実際に確認した。

**実際にメール通知が届くところまで確認できた。**

テスト終了後は、

``` bash
pkill yes
```

でCPU負荷を停止した。

------------------------------------------------------------------------

## 17. Systems Manager Session Manager

Private Subnet内のEC2へ接続するためSession Managerを使用した。

### IAM Role

`mission5-ec2-ssm-role`

### IAM Policy

`AmazonSSMManagedInstanceCore`

EC2へIAMロールを付与し、Session ManagerからPrivate
Subnet内のEC2へ接続できることを確認した。

IAMロール付与直後はSSM
Agentが認証情報を認識できないケースがあり、更新またはEC2再起動によって接続可能になることも確認した。

------------------------------------------------------------------------

## 18. S3 + Lambda

S3イベントをトリガーとしてLambdaを実行するサーバーレス構成を構築。

### S3

Bucket：

`mission5-serverless-bucket-260822`

S3へのObjectCreatedイベントをLambdaのトリガーにした。

### Lambda

S3イベントから、

-   バケット名
-   アップロードされたファイル名

を取得し、CloudWatch Logsへ出力した。

``` text
S3
 ↓ ObjectCreated
Lambda
 ↓
S3イベントを受信
 ↓
バケット名を取得
 ↓
ファイル名を取得
 ↓
CloudWatch Logs
```

実際に画像ファイルおよび `test2.txt`
をアップロードし、Lambdaが実行されることを確認。

CloudWatch Logsで、

``` text
S3からイベントを受信しました
バケット: mission5-serverless-bucket-260822
アップロードされたファイル: test2.txt
```

というログを確認した。

------------------------------------------------------------------------

## 19. Lambda + CloudWatch Logs

Lambda実行時のSTART / END /
REPORTログに加え、S3イベントの内容がCloudWatch
Logsへ記録されることを確認した。

これにより、

``` text
S3 → Lambda → CloudWatch Logs
```

というイベント駆動型サーバーレス処理を実際に動かした。

------------------------------------------------------------------------

## 20. Mission 5で学んだこと

### VPC / ネットワーク

-   Public SubnetとPrivate Subnet
-   Route TableとSubnetの関連付け
-   Internet Gateway
-   NAT Gateway
-   Private Subnetからの外向き通信

### EC2 / Auto Scaling

-   AMI
-   起動テンプレート
-   Auto Scaling
-   Desired / Minimum / Maximum
-   Target Tracking Policy

### ALB

-   Internet-facing ALB
-   Target Group
-   Health Check
-   ALB → EC2の通信

### RDS

-   DB Subnet Group
-   Private Subnetへの配置
-   Security GroupによるDBアクセス制御

### CloudWatch / SNS

-   メトリクス
-   Alarm
-   CPU使用率監視
-   SNS Topic
-   Email Subscription
-   実負荷テスト

### Serverless

-   S3イベント
-   Lambda
-   CloudWatch Logs
-   イベント駆動型アーキテクチャ

### IAM / Systems Manager

-   EC2 IAM Role
-   `AmazonSSMManagedInstanceCore`
-   Session Manager
-   Private SubnetのEC2へSSHなしで接続する方法

------------------------------------------------------------------------

## 21. 完成チェックリスト

  項目                       状態
  -------------------------- -------------
  VPC                        ✅
  Public Subnet ×2           ✅
  Private Subnet ×2          ✅
  Internet Gateway           ✅
  NAT Gateway ×2             ✅
  Route Table                ✅
  Security Group             ✅
  RDS                        ✅
  ベースEC2                  ✅
  Nginx                      ✅
  起動テンプレート           ✅
  Target Group               ✅
  ALB                        ✅
  Auto Scaling Group         ✅
  EC2 ×2                     ✅
  ALB経由Webアクセス         ✅
  CloudWatch                 ✅
  CloudWatch Alarm           ✅
  SNS                        ✅
  メール通知                 ✅ 実証済み
  SSM Session Manager        ✅ 実証済み
  S3                         ✅
  Lambda                     ✅
  S3 → Lambda                ✅ 実証済み
  Lambda → CloudWatch Logs   ✅ 実証済み
  CPU負荷テスト              ✅ 実施済み
  Alarm → SNS → Email        ✅ 実証済み

------------------------------------------------------------------------

## 22. Mission 5の成果

Mission
5では、AWSサービスを単体で触るだけではなく、以下のような**実際のシステムとしての連携**を構築・検証した。

``` text
          Webシステム
              |
        +-----+-----+
        |           |
       ALB         RDS
        |
       ASG
      /       EC2   EC2


       監視・通知
          |
      CloudWatch
          |
        Alarm
          |
         SNS
          |
        Email


      サーバーレス
          |
          S3
          |
       Lambda
          |
   CloudWatch Logs
```

特に、**実際にCPU負荷を発生させてメール通知が届くところまで確認できたこと**により、監視設定が実際に機能することを検証できた。

------------------------------------------------------------------------

## 23. Mission 6

Mission 5ではAPI Gatewayを構築せず、Mission 6へ持ち越す。

### Mission 6予定

**API Gateway + LambdaによるサーバーレスAPI構築**

``` text
User / Browser
       |
       v
API Gateway
       |
       v
Lambda
       |
       v
HTTP Response
```

Mission 6では、API GatewayをHTTP
APIの入口として利用し、Lambdaとの連携を実際に構築・動作確認する。
