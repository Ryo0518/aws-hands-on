# Mission 4 - ALB + Auto Scaling を利用したWebシステム構築

## 1. シナリオ

社内向けWebサービスの利用者増加に備え、可用性と拡張性を考慮したAWS環境を構築する。

WebサーバーをPrivate Subnetに配置し、インターネットからのアクセスはApplication Load Balancer（ALB）経由に限定する。

また、Auto Scaling Group（ASG）を利用して、アクセス負荷に応じてWebサーバーを自動的に増減させる。

データベースにはAmazon RDS for MySQLを利用する。

---

## 2. 要件

### ネットワーク

- VPCを1つ作成
- Availability Zoneを2つ利用
- Public Subnetを2つ作成
- Private Subnetを2つ作成
- Internet Gatewayを利用
- NAT Gatewayを利用
- Public / PrivateそれぞれにRoute Tableを設定

### Webサーバー

- EC2をPrivate Subnetに配置
- Amazon Linux 2023を使用
- nginxをWebサーバーとして利用
- Public IPは付与しない
- AWS Systems Manager Session Managerから管理

### ロードバランサー

- Application Load Balancer（ALB）を利用
- Public Subnetに配置
- Internet-facing
- HTTP : 80で受け付け
- Target Groupを利用してEC2へ転送

### Auto Scaling

- Auto Scaling Groupを利用
- 最小台数：2
- 希望台数：2
- 最大台数：4
- 2つのAZにEC2を分散
- Target Tracking Scaling Policyを利用
- 平均CPU使用率50%をターゲットとする

### データベース

- Amazon RDS for MySQL
- Private Subnetに配置
- DB Subnet Groupを利用
- パブリックアクセス：なし
- 今回は無料利用枠の制約によりSingle-AZで構築

---

## 3. アーキテクチャ

```text
                         Internet
                            │
                            ▼
                  ┌─────────────────┐
                  │ Internet Gateway │
                  └────────┬────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │      ALB        │
                  │     ALB-SG      │
                  │  Public Subnet  │
                  └────────┬────────┘
                           │ HTTP:80
                           ▼
                  ┌─────────────────┐
                  │  Target Group   │
                  └────────┬────────┘
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
       ┌─────────────┐           ┌─────────────┐
       │    EC2      │           │    EC2      │
       │   AZ-a      │           │   AZ-c      │
       │   Private   │           │   Private   │
       │   EC2-SG    │           │   EC2-SG    │
       └──────┬──────┘           └──────┬──────┘
              │                         │
              └────────────┬────────────┘
                           │
                      MySQL :3306
                           ▼
                    ┌─────────────┐
                    │     RDS     │
                    │   MySQL     │
                    │   RDS-SG    │
                    │  Single-AZ  │
                    └─────────────┘

Private EC2
    │
    ▼
NAT Gateway
    │
    ▼
Internet Gateway
    │
    ▼
Internet / AWS Services