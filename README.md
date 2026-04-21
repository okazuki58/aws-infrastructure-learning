# AWS Infrastructure Learning

AWSインフラを体系的に学習するためのリポジトリ。
Phase 1は実際のプロダクション環境（DandoriAI）の構成をベースに各サービスの仕組みを学び、Phase 2はAWS SAA（Solutions Architect Associate）試験対策を行う。

## 背景

DandoriAIのインフラ構成:
- ECS Fargate（アプリケーション）
- ALB（ロードバランサー）
- RDS PostgreSQL（データベース）
- S3（ファイルストレージ）
- SQS + Lambda（非同期ジョブ）
- SSM Parameter Store（設定管理）
- Terraform（IaC）

## 構成

- **Phase 1（Day 1-15）**: 基礎編 - DandoriAIの実体験ベース → :white_check_mark: 完了
- **Phase 2（Day 16-30）**: SAA対策編 - 試験の出題範囲を網羅

## Phase 1: 基礎編

### AWSの全体像とネットワーク基礎（Day 1-3）

| Day | トピック | Status | Issue |
|-----|---------|--------|-------|
| 1 | AWSの全体像 - リージョン・AZ・VPC・サブネット | :white_check_mark: 完了 | #1 |
| 2 | セキュリティグループとIAM - 誰が何にアクセスできるか | :white_check_mark: 完了 | #2 |
| 3 | ネットワーク実践 - DandoriAIのVPC構成を読み解く | :white_check_mark: 完了 | #3 |

### コンピュート（Day 4-6）

| Day | トピック | Status | Issue |
|-----|---------|--------|-------|
| 4 | ECS/Fargate基礎 - タスク定義・サービス・クラスター | :white_check_mark: 完了 | #4 |
| 5 | ALB基礎 - ターゲットグループ・リスナー・ヘルスチェック | :white_check_mark: 完了 | #5 |
| 6 | ECS運用 - デプロイ・スケーリング・トラブルシューティング | :white_check_mark: 完了 | #6 |

### ストレージとデータ（Day 7-9）

| Day | トピック | Status | Issue |
|-----|---------|--------|-------|
| 7 | S3基礎 - バケット・オブジェクト・アクセス制御 | :white_check_mark: 完了 | #7 |
| 8 | RDS基礎 - インスタンス・スキーマ分離・バックアップ | :white_check_mark: 完了 | #8 |
| 9 | SSM Parameter Store - 設定管理とシークレット | :white_check_mark: 完了 | #9 |

### 非同期処理（Day 10-11）

| Day | トピック | Status | Issue |
|-----|---------|--------|-------|
| 10 | SQS基礎 - キュー・メッセージ・DLQ | :white_check_mark: 完了 | #10 |
| 11 | Lambda基礎 - 関数・トリガー・環境変数 | :white_check_mark: 完了 | #11 |

### IaCとCI/CD（Day 12-14）

| Day | トピック | Status | Issue |
|-----|---------|--------|-------|
| 12 | Terraform基礎 - state・plan・apply・モジュール | :white_check_mark: 完了 | #12 |
| 13 | Terraform実践 - DandoriAIのインフラコードを読む | :white_check_mark: 完了 | #13 |
| 14 | CI/CD - GitHub Actions + ECR + ECSデプロイ | :white_check_mark: 完了 | #14 |

### 総合演習（Day 15）

| Day | トピック | Status | Issue |
|-----|---------|--------|-------|
| 15 | 総合演習 - 障害対応シミュレーション | :white_check_mark: 完了 | #15 |

## Phase 2: SAA対策編

### 安全なアーキテクチャ（Day 16-18）

| Day | トピック | Status | Issue |
|-----|---------|--------|-------|
| 16 | IAM応用 - ユーザー・グループ・フェデレーション・MFA | :red_circle: 未着手 | #16 |
| 17 | 暗号化 - KMS・Secrets Manager・CloudHSM | :red_circle: 未着手 | #17 |
| 18 | ネットワークセキュリティ - WAF・Shield・GuardDuty | :red_circle: 未着手 | #18 |

### 弾力性・可用性（Day 19-21）

| Day | トピック | Status | Issue |
|-----|---------|--------|-------|
| 19 | 高可用性パターン - Multi-AZ・リージョン冗長 | :red_circle: 未着手 | #19 |
| 20 | Auto Scaling詳細 - EC2/ECS/DynamoDB | :red_circle: 未着手 | #20 |
| 21 | バックアップ・DR戦略 | :red_circle: 未着手 | #21 |

### 高パフォーマンス（Day 22-24）

| Day | トピック | Status | Issue |
|-----|---------|--------|-------|
| 22 | コンピュート選択 - EC2/ECS/EKS/Lambda/Fargate | :red_circle: 未着手 | #22 |
| 23 | データベース選択 - RDS/Aurora/DynamoDB/ElastiCache | :red_circle: 未着手 | #23 |
| 24 | コンテンツ配信 - CloudFront・Global Accelerator | :red_circle: 未着手 | #24 |

### コスト最適化（Day 25-27）

| Day | トピック | Status | Issue |
|-----|---------|--------|-------|
| 25 | EC2価格モデル - On-Demand/Reserved/Savings Plans/Spot | :red_circle: 未着手 | #25 |
| 26 | ストレージ最適化 - S3ストレージクラス・ライフサイクル | :red_circle: 未着手 | #26 |
| 27 | データ転送コスト - VPC Peering/PrivateLink/Endpoint | :red_circle: 未着手 | #27 |

### 統合・分析・メッセージング（Day 28-29）

| Day | トピック | Status | Issue |
|-----|---------|--------|-------|
| 28 | メッセージング - SQS vs SNS vs EventBridge | :red_circle: 未着手 | #28 |
| 29 | 分析・ETL - Kinesis・Glue・Athena・Redshift | :red_circle: 未着手 | #29 |

### 総仕上げ（Day 30）

| Day | トピック | Status | Issue |
|-----|---------|--------|-------|
| 30 | 総仕上げ - 過去問演習 + 模擬試験 | :red_circle: 未着手 | #30 |

## 進捗
- 開始日: 2026-04-05
- 完了Day: 15 / 30
- 総学習時間: 0h

## 学習ログ
各Dayの学習記録はIssueコメントおよび `days/dayXX/notes.md` に蓄積される。
