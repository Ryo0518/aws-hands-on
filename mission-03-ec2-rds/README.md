Mission 3：EC2 + RDSによる社員情報管理システム

1. シナリオ・要件

社内向けの社員情報管理システムを想定し、AWS上にWebサーバーとデータベースを分離した構成を構築する。

要件

* Webサーバー：Amazon EC2
* データベース：Amazon RDS for MySQL
* EC2はPublic Subnetに配置
* RDSはPrivate Subnetに配置
* RDSはインターネットから直接アクセスできない構成
* Security GroupによってEC2からRDSへのMySQL通信のみ許可
* EC2からRDSへ実際に接続できることを確認する

⸻

2. アーキテクチャ

ネットワーク構成

* VPC：10.0.0.0/16
* Public Subnet：10.0.1.0/24
* Private Subnet：10.0.2.0/24
* Private Subnet 2：10.0.3.0/24
* Private Subnetは異なるAZに配置
* Internet GatewayをVPCにアタッチ
* Public Subnet用Route TableからInternet Gatewayへルーティング

RDS

* Engine：MySQL
* DB Subnet Group：mission3-db-subnet-group
* Private Subnetを2つのAZから登録
* Public access：No

⸻

3. Security Group

EC2 Security Group

mission3-ec2-SG

Inbound：

Protocol	Port	Source
SSH	22	自分のIP
HTTP	80	Internet

RDS Security Group

mission3-RDS-SG

Inbound：

Protocol	Port	Source
MySQL	3306	mission3-ec2-SG

RDSの3306番ポートは、IPアドレスではなくEC2のSecurity Groupを送信元として許可した。

これにより、EC2からのMySQL通信のみを許可する構成とした。

⸻

4. 動作確認

EC2へのSSH接続

EC2にSSH接続し、Amazon Linux 2023上で操作できることを確認。

RDSへのネットワーク疎通

EC2にnmap-ncatをインストールし、RDSの3306番ポートへの接続を確認。

nc -zv <RDSエンドポイント> 3306

以下のような成功メッセージを確認。

Ncat: Connected to 10.0.2.168:3306.

RDSへのMySQL接続

MariaDB/MySQLクライアントをEC2にインストール。

sudo dnf install mariadb105

RDSへ接続。

mysql -h <RDSエンドポイント> -P 3306 -u admin -p

EC2からRDSのMySQLへログインできることを確認。

⸻

5. データベース操作

RDS上にemployee_dbデータベースを作成。

CREATE DATABASE employee_db;
USE employee_db;

社員情報を格納するemployeesテーブルを作成。

CREATE TABLE employees (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    department VARCHAR(100)
);

データを登録。

INSERT INTO employees (id, name, department)
VALUES (1, 'Tanaka', 'Human Resources');
INSERT INTO employees (id, name, department)
VALUES (2, 'Sato', 'IT');
INSERT INTO employees (id, name, department)
VALUES (3, 'Suzuki', 'General Affairs');

登録したデータを確認。

SELECT * FROM employees;

WHEREを使用した条件検索も実施。

SELECT name, department
FROM employees
WHERE department = 'IT';

⸻

6. 学んだこと

VPC / Subnet

* VPCの中にSubnetを作成する
* Public SubnetとPrivate Subnetは、ルーティングによってインターネットへの到達性が変わる
* RDSでは複数AZのSubnetを使用してDB Subnet Groupを構成する

Internet Gateway / Route Table

Public Subnetからインターネットへ通信するためには、Route TableからInternet Gatewayへのルートが必要。

0.0.0.0/0 → Internet Gateway

Security Group

Security Groupは通信を制御する仮想ファイアウォール。

今回、RDSの3306番ポートについて、

mission3-ec2-SG
        ↓
      TCP 3306
        ↓
mission3-RDS-SG

というSecurity Group間の許可を設定した。

EC2とRDSの役割分担

EC2をWebサーバー、RDSをデータベースとして分離することで、サーバーとデータベースの役割を分離できる。

RDSのPrivate配置

RDSをPublicアクセスさせず、EC2からのみアクセスできる構成にすることで、データベースを直接インターネットへ公開しない構成を実現した。

SQL

基本的なSQL操作を体験した。

* CREATE DATABASE
* CREATE TABLE
* INSERT
* SELECT
* WHERE

⸻

7. トラブルシューティング

SSH接続できない

EC2作成時にキーペアを設定し忘れたため、EC2を作り直した。

また、秘密鍵の保存場所を指定せずSSH接続したため、

Identity file ... not accessible

が発生。

秘密鍵のパスを明示することで解決。

ssh -i ~/Downloads/mission3-keypair.pem ec2-user@<Public IP>

RDSのSecurity Groupが選択肢に表示されない

RDSがMission 3用VPCとは異なるVPCに存在していたため、Mission 3用Security Groupが選択できなかった。

DB Subnet GroupをMission 3用に変更し、RDSをMission 3用VPCへ移動することで解決。

ncコマンドが存在しない

Amazon Linux 2023にncがインストールされていなかった。

dnf search netcat

でパッケージを検索し、nmap-ncatをインストール。

MySQL接続コマンドのオプションミス

以下のコマンドでは、-pと3306の位置を誤った。

mysql -h <endpoint> -p 3306 -u admin -p

正しくは、

mysql -h <endpoint> -P 3306 -u admin -p

-Pはポート番号、-pはパスワード入力を意味する。

また、-u adminを省略するとOSユーザー名が使用されるため、MySQL認証に失敗した。

⸻

8. Mission 3の成果

AWSコンソールを使用して、

VPC → Subnet → Route Table → Internet Gateway → Security Group → EC2 → RDS

という一連のAWSリソースを構築した。

さらに、

EC2 → RDS → MySQL → SQL

まで実際に動作確認し、AWS上でWebサーバーとデータベースを分離した基本的なシステム構成を構築できた。