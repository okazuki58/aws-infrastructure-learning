# Day 14: CI/CD - GitHub Actions + ECR + ECSデプロイ

- 学習日: 2026-04-21
- 所要時間: 約25分

## セクション1: CI/CDとは

### CIとCD
```
CI（Continuous Integration）:
  コードをpushするたびに自動でテスト・ビルド

CD（Continuous Deployment/Delivery）:
  テストが通ったら自動でデプロイ
```

### DandoriAIではGitHub Actionsが担当
- GitHubリポジトリのイベントに反応して自動でコードを実行
- `.github/workflows/*.yml` にワークフローを記述

### DandoriAIのワークフロー一覧
```
github-action-deploy.yml     → 本番デプロイ
github-action-deploy-stg.yml → ステージングデプロイ
create-release-pr-prod.yml   → リリースPRの自動作成
create-release-pr-stg.yml    → ステージングリリースPR
create-issue-from-file.yml   → Issue自動作成
```

---

## セクション2: 本番デプロイの流れ

```
[PRマージ]
  ↓
[developブランチにpush]
  ↓
[GitHub Actions起動]
  ↓
[Dockerイメージをビルド]
  Dockerfile.prodを使う
  コミットハッシュでタグ付け（例: main-d3db09e）
  ↓
[ECRにpush]
  ↓
[ECSタスク定義を新しいイメージで更新]
  revision番号が増える（例: 27 → 28）
  ↓
[ECSサービスをforce-new-deployment]
  ローリングデプロイで無停止入れ替え
  ↓
[Lambda関数のコードを更新]
  short/medium/long の3つすべて更新
  ↓
[デプロイ完了]
```

---

## セクション3: 用語整理

### revision
- ECSタスク定義のバージョン番号
- 設定（CPU、メモリ、イメージ等）を変更するたびに新しいrevisionになる
- 例: revision 6, 7, 8... と増えていく

### タグ
- Dockerイメージの名札
- どのバージョンのイメージか識別するため
- 例: `main-d3db09e` = mainブランチのコミット `d3db09e` からビルドされたイメージ

### ECR（Elastic Container Registry）
- Dockerイメージの倉庫（AWSサービス）
- ビルドしたイメージをpushして保管
- ECSがタスク起動時にここからpullする

### ローリングデプロイ
- 新旧タスクを徐々に入れ替える方式
- 無停止でデプロイできる
- Day 4-5で学んだ内容

---

## セクション4: 前回のトラブルとの対応

### revision 7のdummy問題
```
revision 7のイメージ: ...:dummy
→ ECRに "dummy" というタグのイメージは存在しない
→ pullできない → CannotPullContainerError
→ タスクが起動できない
```

### CI/CDのタイムアウト
```
原因: npm ci が30分以内に終わらなかった
  - busboy, @aws-sdk/lib-storage, yauzl を新たに追加
  - Dockerレイヤーキャッシュが無効化
  - 依存の再ダウンロードで30分オーバー

対処: Re-run
  - 2回目はキャッシュが効いて完了
  - デプロイ成功
```

### 教訓
- 依存を大量に追加するとCI時間が伸びる
- CIタイムアウト設定（30分）は初回は足りない可能性がある

---

## セクション5: CI/CDでやること・やらないこと

### CI/CDでやること（自動化）
- コードのビルド
- Dockerイメージのpush
- ECSタスク定義の更新
- Lambdaコードの更新

### CI/CDでやらないこと（手動）
- インフラの変更（Terraform apply）
  → dandori-infraリポジトリの変更は手動でapply
- SSMパラメータの変更
  → `aws ssm put-parameter` で手動
- ECSの再起動（SSM変更時など）
  → `aws ecs update-service` で手動

### ポイント
**コード変更はCI/CD、インフラ変更は手動** という使い分け

---

## 復習ポイント
- [ ] CIとCDの違いを説明できるか？
- [ ] PRマージから本番デプロイまでの流れを説明できるか？
- [ ] revisionとタグの違いを説明できるか？
- [ ] ECRの役割を説明できるか？
- [ ] 前回のrevision 7のdummy問題をCI/CDの観点で説明できるか？
- [ ] CI/CDでやることとやらないことを区別できるか？
