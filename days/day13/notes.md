# Day 13: Terraform実践 - DandoriAIのインフラコードを読む

- 学習日: 2026-04-21
- 所要時間: 約25分

## セクション1: 全体構造

### 3階層に分かれている
```
dandori-infra/
├── environments/       ← 「このインフラを実際に作れ」という指示書
│   ├── poc/           ← POC環境（複数テナント）
│   ├── production/    ← 本番環境
│   └── staging/       ← ステージング環境
│
└── modules/           ← 再利用可能な部品
    └── poc-tenant/    ← POCテナントを作るための部品セット
```

### environmentsとmodulesの役割
```
environments/ = 実際に作るもの
  「POC環境のnhk-epテナントを作る」「production環境を作る」

modules/ = 再利用可能な部品
  「POCテナントを作るための部品」
```

---

## セクション2: POC環境の構成

### shared vs tenants
```
environments/poc/
├── shared/    ← 全テナントで共有するリソース（ALB、RDS、ECSクラスター等）
├── tenants/   ← 各テナントのリソース（モジュールを呼び出す）
├── docs/      ← ドキュメント（TENANT_SETUP.md等）
├── scripts/   ← 運用スクリプト（sync-ssm-params.sh等）
└── Makefile   ← よく使うコマンドのショートカット
```

### なぜ2層に分けるか
- **applyの影響範囲を絞るため**
- テナントの変更 → tenants/だけapply → sharedは触らない
- 間違ってRDSなど重要リソースを壊すリスクを減らす

### stateも別々
```
dandori-terraform-state-bucket/
├── poc/shared/         ← shared用のstate
└── poc/tenants/        ← tenants用のstate
```

---

## セクション3: テナント定義

### variables.tfでテナントを定義
```hcl
tenants = {
  "nhk-ep" = {
    display_name = "株式会社NHK"
    fqdn         = "nhk-ep.dandori.xincere.jp"
    priority     = 10
  }
  "odakyu-ag" = { ... }
  "demo"     = { ..., demo_mode = true }
}
```

### main.tfでfor_eachで展開
```hcl
module "tenant" {
  source   = "../../../modules/poc-tenant"
  for_each = local.enabled_tenants

  tenant_key   = each.key
  display_name = each.value.display_name
}
```

### 動作
- マップの各エントリごとにモジュールを呼び出す
- 3テナント → 3つのECS、3つのSQS、3つのS3等が作られる

### テナント追加が簡単な理由
variables.tfに1行追加するだけで、そのテナント用のリソース群がまとめて作られる。

---

## セクション4: モジュールの中身

### poc-tenant モジュール
```
modules/poc-tenant/
├── ecs_app.tf             → Next.jsアプリのECS設定
├── ecs_weaviate.tf        → WeaviateのECS設定
├── ingress.tf             → ALBリスナールール、Route53設定
├── lambda_sqs_processor.tf → Lambdaの3関数（short/medium/long）
├── queues.tf              → SQSキューの3つ
├── storage.tf             → S3バケット
├── database.tf            → RDSスキーマとユーザー
├── parameters.tf          → SSMパラメータ群
├── backups.tf             → バックアップ設定
├── main.tf                → 共通のlocals
├── variables.tf           → このモジュールが受け取る変数
└── outputs.tf             → このモジュールが返す値
```

### 前回のトラブルとの対応
| やったこと | 関係するファイル |
|---|---|
| ALBのidle_timeout変更 | `shared/ingress.tf` |
| Lambda IAM修正 | `shared/iam.tf` |
| S3_BUCKET設定ミス | `parameters.tf`（lifecycle ignore_changesのせい） |
| SQSリトライ権限 | `shared/iam.tf` |
| テナントのドメイン変更 | `tenants/variables.tf` |

---

## セクション5: .envrcによる環境変数管理

### 仕組み
```bash
# environments/poc/.envrc
export TF_VAR_project_name=dandori
export TF_VAR_aws_region=ap-northeast-1
export TF_VAR_app_port=3000
```

### 動作
- `TF_VAR_xxx`という環境変数が、Terraformの`variable "xxx"`に自動的に渡される
- 環境ごとに違う値を外から渡せる
- コードは同じでも、POCは`dandori-poc-*`、productionは`dandori-production-*`になる

---

## セクション6: テナント追加の流れ

### ドキュメント: TENANT_SETUP.md
```
1. tenants/variables.tf にテナント定義を追加
2. make apply-tenants
   → ECSサービス、SQS、Lambda、S3、IAMが自動作成
3. sync-ssm-params.sh でパラメータをコピー
4. ドメイン依存のパラメータを手動修正
5. S3_BUCKET も手動修正（前回の教訓）
6. make apply-tenants で Lambda 環境変数を反映
```

### 前回のトラブル
- Step 5のS3_BUCKET修正を忘れると、他テナントのバケットを参照してしまう
- sync-ssm-params.shがS3_BUCKETも上書きしてしまうため

---

## 復習ポイント
- [ ] environmentsとmodulesの役割の違いを説明できるか？
- [ ] sharedとtenantsを分ける理由を説明できるか？
- [ ] for_eachでテナント展開する仕組みを説明できるか？
- [ ] モジュール内のファイルがどう機能ごとに分かれているか説明できるか？
- [ ] TF_VAR_xxxの環境変数が何に使われるか説明できるか？
- [ ] テナント追加の手順を説明できるか？
