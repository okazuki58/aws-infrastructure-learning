# Day 9: SSM Parameter Store - 設定管理とシークレット

- 学習日: 2026-04-16
- 所要時間: 約25分

## セクション1: SSM Parameter Storeとは

### 設定値の外部保管庫
```
SSMがない場合:
  設定値をコードに直書き or .envファイル
  → 環境ごとに値を変えるにはコード変更が必要
  → APIキーがコードに入ってしまう

SSMがある場合:
  設定値をAWSに保管し、アプリ起動時に取得
  → 同じコードで環境ごとに異なる設定で動く
  → APIキーはAWSに安全に保管
```

### 確認コマンド
```bash
# テナントの全パラメータ一覧
aws ssm get-parameters-by-path --path "/dandori-poc-odakyu-ag" \
  --query "Parameters[*].{Name:Name,Type:Type}" --output table

# 特定パラメータの値を確認
aws ssm get-parameter --name "/dandori-poc-odakyu-ag/S3_BUCKET" \
  --query "Parameter.Value" --output text

# 値を変更
aws ssm put-parameter --name "/dandori-poc-odakyu-ag/S3_BUCKET" \
  --value "新しい値" --type String --overwrite
```

---

## セクション2: String vs SecureString

### 違い
```
String:
  → 平文で保存。値がそのまま見える
  → 秘密ではない設定値に使う
  → 例: S3_BUCKET, NODE_ENV, AI_MODEL_ESTIMATE

SecureString:
  → KMS（AWS Key Management Service）で暗号化して保存
  → APIキーやパスワードなど漏洩したらまずいものに使う
  → 例: ANTHROPIC_API_KEY, SENDGRID_API_KEY, BETTER_AUTH_SECRET
```

### 使い分けの基準
- バケット名、環境名、モデル名 → String（漏れても大丈夫）
- APIキー、シークレット、パスワード → SecureString（漏れたらまずい）

---

## セクション3: パラメータの階層構造

### 命名規則
```
/プロジェクト名-環境-テナント/パラメータ名

例:
/dandori-poc-odakyu-ag/S3_BUCKET
/dandori-poc-nhk-ep/S3_BUCKET
/dandori-poc-demo/S3_BUCKET
```

### 階層構造のメリット
- テナントごとにパラメータを分離できる
- プレフィックスで一括取得できる（get-parameters-by-path）
- IAMポリシーでワイルドカード指定できる（`/dandori-poc-*`）

### 実体験: sync-ssm-params.shの問題
```
プレフィックスを置換してコピー:
  /dandori-poc-nhk-ep/S3_BUCKET → /dandori-poc-odakyu-ag/S3_BUCKET
  ただし値（dandori-poc-nhk-ep-bucket）はそのまま → バグの原因！
```

---

## セクション4: ECSとLambdaでの読み込み方の違い

### ECS（secrets経由）
```
タスク定義にSSMのARNを記載
→ 起動時にECSがSSMから値を取得して環境変数にセット
→ SSM変更後はECS再起動（force-new-deployment）が必要
```

### Lambda（Terraform経由）
```
TerraformがSSMの値を読み取ってLambdaの環境変数に直接書き込む
→ SSM変更後はterraform apply（make apply-tenants）が必要
```

### ポイント
- ECSもLambdaも、SSMを変更しただけでは反映されない
- ECS: 再起動が必要
- Lambda: terraform applyが必要

---

## セクション5: Terraformとlifecycle ignore_changes

### 仕組み
```terraform
resource "aws_ssm_parameter" "s3_bucket" {
  name  = "${local.ssm_prefix}/S3_BUCKET"
  value = aws_s3_bucket.tenant.bucket
  lifecycle { ignore_changes = [value] }
}
```

### 動作
```
初回terraform apply:
  → S3バケットを作成し、バケット名をSSMに自動設定（正しい値）

2回目以降のterraform apply:
  → SSMの値が変わっていても無視（上書きしない）
  → 手動で変更した値を保持する
```

### 実体験: この仕組みが裏目に出たケース
```
1. terraform apply → S3_BUCKET = "dandori-poc-odakyu-ag-bucket"（正しい）
2. sync-ssm-params.sh → S3_BUCKET = "dandori-poc-nhk-ep-bucket"（上書き）
3. terraform apply → ignore_changesなので修正されない（放置）
→ 手動で aws ssm put-parameter で修正して対処
```

---

## 復習ポイント
- [ ] SSM Parameter Storeの役割を説明できるか？
- [ ] StringとSecureStringの違いと使い分けを説明できるか？
- [ ] パラメータの階層構造の命名規則を説明できるか？
- [ ] ECSとLambdaでSSMの読み込み方が違う理由を説明できるか？
- [ ] SSMの値を変更した後、ECSとLambdaそれぞれで何が必要か説明できるか？
- [ ] lifecycle ignore_changesの意味とメリット・デメリットを説明できるか？
- [ ] S3_BUCKETの設定ミスがなぜ起きて、なぜterraform applyで直らなかったか説明できるか？
