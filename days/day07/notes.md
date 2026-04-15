# Day 7: S3基礎 - バケット・オブジェクト・アクセス制御

- 学習日: 2026-04-16
- 所要時間: 約25分

## セクション1: S3の基本概念

### 本棚の例え
```
バケット = 本棚（1つの棚）
  例: dandori-poc-odakyu-ag-bucket

オブジェクト = 本（ファイル1つ1つ）
  例: 見積書PDF、議事録テキスト

キー = 本棚の中の位置（パス）
  例: 3c7b95f4.../learning-temp-files/abc123/estimate.pdf
```

### 「フォルダ」は存在しない
- S3にフォルダという概念はない
- キーに`/`を含めることで見た目上フォルダに見えているだけ
- `3c7b95f4/learning-temp-files/session1/file.pdf` はただの1つの文字列

### DandoriAIのバケット構成
```
POC環境（テナントごとにバケット分離）:
  dandori-poc-nhk-ep-bucket
  dandori-poc-odakyu-ag-bucket
  dandori-poc-demo-bucket

Production環境（1つのバケット）:
  dandori-production-bucket

その他:
  dandori-production-alb-log     ← ALBアクセスログ
  dandori-terraform-state-bucket ← Terraformの状態管理
```

### 確認コマンド
```bash
# バケット一覧
aws s3 ls

# バケットの中身
aws s3 ls s3://dandori-poc-odakyu-ag-bucket/

# 特定プレフィックス配下
aws s3 ls s3://dandori-poc-odakyu-ag-bucket/3c7b95f4.../
```

---

## セクション2: キー設計とデータ分離

### DandoriAIのキー設計ルール
```
{companyId}/{用途}/{詳細}

例:
3c7b95f4.../ （会社ID）
├── learning-source-files/  ← 学習データ（確定保存）
├── learning-temp-files/    ← 学習データ（一時アップロード）
├── source-docs-files/      ← 議事録ファイル
└── source-docs/            ← 議事録データ
```

### プレフィックスによるデータ分離
- オブジェクトキーは必ず`{companyId}/`から始める
- `{companyId}/`プレフィックスで会社のデータを一括削除可能
- POCではバケットも分離、productionではプレフィックスで分離

### S3操作コマンド
```bash
# ファイルを削除（1つ）
aws s3 rm s3://バケット名/キー

# プレフィックス配下を一括削除
aws s3 rm s3://バケット名/プレフィックス/ --recursive
```

---

## セクション3: アクセス制御

### IAMによるS3アクセス制御
```
ECSタスクロール:
  Action: s3:GetObject, s3:PutObject, s3:ListBucket, s3:DeleteObject
  Resource: "arn:aws:s3:::dandori-poc-*"

Lambdaロール:
  Action: s3:GetObject, s3:PutObject, s3:ListBucket
  Resource: "arn:aws:s3:::dandori-poc-*"
```

### ワイルドカードのトレードオフ
- `dandori-poc-*` で全テナントのバケットにアクセス可能
- メリット: テナント追加時にIAMポリシー変更不要
- デメリット: SSM設定ミスでテナント間のデータ混入をIAMで防げない

### 実体験: S3_BUCKETの設定ミス
```
SSMの S3_BUCKET が nhk-ep のバケットを指していた
→ odakyu-ag のデータが nhk-ep のバケットに入ってしまった
→ IAMはワイルドカードなのでエラーにならず気づけなかった
→ SSMパラメータを修正して対処
```

---

## セクション4: マルチパートアップロード

### 通常アップロード vs マルチパートアップロード
```
通常（PutObject）:
  1回のリクエストでファイル全体を送る
  → 小さいファイル向け（数MB以下）
  → 途中で失敗したら最初からやり直し

マルチパート:
  ファイルを分割して複数リクエストで送る
  → 大きいファイル向け（数MB以上）
  → 途中で失敗してもその部分だけやり直し
  → 並列で送れるので速い
```

### DandoriAIでの実装
```typescript
const upload = new Upload({
  partSize: 8 * 1024 * 1024,  // 8MBずつ分割
  queueSize: 1,                // 同時アップロード数1（メモリ節約）
});
```
- 350MBのZIP → 8MBずつ約44回に分割
- queueSize=1で順番に送るのでメモリには8MBしか載らない
- @aws-sdk/lib-storage の Upload クラスが内部的にマルチパートを使用

---

## 復習ポイント
- [ ] バケット・オブジェクト・キーの関係を説明できるか？
- [ ] S3にフォルダが存在しない理由を説明できるか？
- [ ] DandoriAIのキー設計ルール（{companyId}/から始める）の理由を説明できるか？
- [ ] ワイルドカードのIAMポリシーのメリット・デメリットを説明できるか？
- [ ] 通常アップロードとマルチパートアップロードの違いを説明できるか？
- [ ] S3_BUCKETの設定ミスがなぜIAMでブロックされなかったか説明できるか？
