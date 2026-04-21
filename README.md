# AWS Infrastructure Learning

AWSインフラを体系的に学習するためのリポジトリ。
実際のプロダクション環境（DandoriAI）の構成をベースに、各サービスの仕組みと運用を学ぶ。

## 背景

DandoriAIのインフラ構成:
- ECS Fargate（アプリケーション）
- ALB（ロードバランサー）
- RDS PostgreSQL（データベース）
- S3（ファイルストレージ）
- SQS + Lambda（非同期ジョブ）
- SSM Parameter Store（設定管理）
- Terraform（IaC）

## カリキュラム

### Phase 1: AWSの全体像とネットワーク基礎（Day 1-3）

| Day | トピック | Status | Issue |
|-----|---------|--------|-------|
| 1 | AWSの全体像 - リージョン・AZ・VPC・サブネット | :white_check_mark: 完了 | #1 |
| 2 | セキュリティグループとIAM - 誰が何にアクセスできるか | :white_check_mark: 完了 | #2 |
| 3 | ネットワーク実践 - DandoriAIのVPC構成を読み解く | :white_check_mark: 完了 | #3 |

### Phase 2: コンピュート（Day 4-6）

| Day | トピック | Status | Issue |
|-----|---------|--------|-------|
| 4 | ECS/Fargate基礎 - タスク定義・サービス・クラスター | :white_check_mark: 完了 | #4 |
| 5 | ALB基礎 - ターゲットグループ・リスナー・ヘルスチェック | :white_check_mark: 完了 | #5 |
| 6 | ECS運用 - デプロイ・スケーリング・トラブルシューティング | :white_check_mark: 完了 | #6 |

### Phase 3: ストレージとデータ（Day 7-9）

| Day | トピック | Status | Issue |
|-----|---------|--------|-------|
| 7 | S3基礎 - バケット・オブジェクト・アクセス制御 | :white_check_mark: 完了 | #7 |
| 8 | RDS基礎 - インスタンス・スキーマ分離・バックアップ | :white_check_mark: 完了 | #8 |
| 9 | SSM Parameter Store - 設定管理とシークレット | :white_check_mark: 完了 | #9 |

### Phase 4: 非同期処理（Day 10-11）

| Day | トピック | Status | Issue |
|-----|---------|--------|-------|
| 10 | SQS基礎 - キュー・メッセージ・DLQ | :white_check_mark: 完了 | #10 |
| 11 | Lambda基礎 - 関数・トリガー・環境変数 | :white_check_mark: 完了 | #11 |

### Phase 5: IaCとCI/CD（Day 12-14）

| Day | トピック | Status | Issue |
|-----|---------|--------|-------|
| 12 | Terraform基礎 - state・plan・apply・モジュール | :white_check_mark: 完了 | #12 |
| 13 | Terraform実践 - DandoriAIのインフラコードを読む | :white_check_mark: 完了 | #13 |
| 14 | CI/CD - GitHub Actions + ECR + ECSデプロイ | :red_circle: 未着手 | #14 |

### Phase 6: 総合演習（Day 15）

| Day | トピック | Status | Issue |
|-----|---------|--------|-------|
| 15 | 総合演習 - 障害対応シミュレーション | :red_circle: 未着手 | #15 |

## 進捗
- 開始日: 2026-04-05
- 完了Day: 13 / 15
- 総学習時間: 0h

## 学習ログ
各Dayの学習記録はIssueコメントに蓄積される。
