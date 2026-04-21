# Phase 1: 基礎編 まとめ

- 期間: 2026-04-05 〜 2026-04-21
- 完了Day数: 15 / 15

## 学習の流れ

Phase 1は「DandoriAIの実体験をベースに、AWSインフラの各サービスを理解する」が目的でした。
前回のセッションで起きたトラブル（OOM Kill、SSMの設定ミス、Lambda IAM不足等）を教材として、
各サービスの仕組みと関連性を学びました。

## Phase構成

| Phase | Day | トピック |
|-------|-----|---------|
| Phase 1: ネットワーク基礎 | 1-3 | VPC・サブネット・SG・IAM・Route53 |
| Phase 2: コンピュート | 4-6 | ECS・Fargate・ALB・トラブルシュート |
| Phase 3: ストレージとデータ | 7-9 | S3・RDS・SSM |
| Phase 4: 非同期処理 | 10-11 | SQS・Lambda |
| Phase 5: IaCとCI/CD | 12-14 | Terraform・GitHub Actions |
| Phase 6: 総合演習 | 15 | 障害対応シミュレーション |

## DandoriAIのアーキテクチャ全体図

```
ユーザー
  ↓ DNS解決（Route53）
  ↓
ALB（パブリックサブネット）
  ├─ リスナー: 80 → HTTPSへリダイレクト / 404
  └─ リスナー: 443
     └─ リスナールール（Hostヘッダーで振り分け）
        ├─ odakyu-ag.dandori.xincere.jp → odakyu-agのTG
        ├─ nhk-ep.dandori.xincere.jp    → nhk-epのTG
        └─ demo.dandori.xincere.jp      → demoのTG

ターゲットグループ（各テナント）
  └─ ECS Fargate（プライベートサブネット）
     └─ Next.jsコンテナ
        ├─ 同期処理 → レスポンス返却
        └─ 非同期処理 → SQSに投入
                        ↓
                        SQS（short/medium/long）
                        ↓
                        Lambda（short/medium/long）
                        ↓
                        処理結果をRDSに保存

バックエンドサービス:
  - RDS PostgreSQL（プライベートサブネット、スキーマで分離）
  - S3（テナントごとのバケット）
  - Weaviate（ベクトルDB）
  - SSM Parameter Store（設定値・APIキー）
```

## 前回セッションのトラブルとPhase 1の対応

| トラブル | 関連Day | 学んだこと |
|---------|--------|-----------|
| 学習データアップロード504エラー | Day 5, 6, 11 | ALB idle_timeout, OOM Kill, トラブルシュート |
| SSMのS3_BUCKET設定ミス | Day 7, 9 | キープレフィックス設計, lifecycle ignore_changes |
| Lambda AWS_ACCOUNT_ID不足 | Day 9, 11 | ECSとLambdaの環境変数の違い |
| Lambda sqs:SendMessage権限不足 | Day 2, 11 | IAMポリシー, Lambdaロール |
| ルールブック15分タイムアウト | Day 11 | Lambdaの制約, SQSでの分割処理 |
| バッチ17で止まる問題 | Day 10 | SQSのVisibility Timeout, DLQ |
| 精度検証のMaximum call stack | - | コードのバグ（循環再帰） |

## Phase 1で身についた知識

### 説明できるようになったこと
- AWSの基本構成要素（VPC、サブネット、AZ、リージョン）の関係
- パブリック/プライベートサブネットの違いと使い分け
- セキュリティグループとIAMの2層のセキュリティ
- ECS/Fargate/ALB/SQS/Lambdaの役割と関係
- マルチテナント分離の方法（S3バケット、RDSスキーマ）
- Terraformの基本フロー（init/plan/apply/state）
- CI/CDとインフラ変更の使い分け

### 調査・運用できるようになったこと
- トラブル時の切り分け手順（大→小）
- CloudWatch Logsでのログ調査
- ECSサービスの状態確認と再起動
- SSMパラメータの確認と変更
- SQSキューの状態確認とパージ
- TerraformのPR作成とapply

## Phase 2（SAA対策編）への接続

Phase 1で土台ができたので、Phase 2では以下を深掘り予定:

- 高可用性・耐障害性設計（Multi-AZ、Auto Scaling、フェイルオーバー）
- コスト最適化（Savings Plans、Spot、S3ストレージクラス）
- セキュリティ（KMS、Secrets Manager、WAF、Shield）
- 広範囲のサービス（DynamoDB、Aurora、ElastiCache、Kinesis、Glue等）
- サービスの使い分け判断（ECS vs EKS、SQS vs SNS vs EventBridge）
- 過去問演習

## 各Day学習記録へのリンク

- [Day 1: AWSの全体像](days/day01/notes.md)
- [Day 2: セキュリティグループとIAM](days/day02/notes.md)
- [Day 3: ネットワーク実践](days/day03/notes.md)
- [Day 4: ECS/Fargate基礎](days/day04/notes.md)
- [Day 5: ALB基礎](days/day05/notes.md)
- [Day 6: ECS運用・トラブルシューティング](days/day06/notes.md)
- [Day 7: S3基礎](days/day07/notes.md)
- [Day 8: RDS基礎](days/day08/notes.md)
- [Day 9: SSM Parameter Store](days/day09/notes.md)
- [Day 10: SQS基礎](days/day10/notes.md)
- [Day 11: Lambda基礎](days/day11/notes.md)
- [Day 12: Terraform基礎](days/day12/notes.md)
- [Day 13: Terraform実践](days/day13/notes.md)
- [Day 14: CI/CD](days/day14/notes.md)
- [Day 15: 総合演習](days/day15/notes.md)
