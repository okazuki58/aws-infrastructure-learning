# Day 12: Terraform基礎 - state・plan・apply・モジュール

- 学習日: 2026-04-16
- 所要時間: 約30分

## セクション1: IaC（Infrastructure as Code）

### IaCの意義
```
IaCなし:
  AWSコンソールで手動でリソース作成
  → 誰が何を作ったか不明
  → 同じ構成の複製ができない
  → 変更履歴が残らない

IaCあり:
  .tfファイルにインフラを記述
  → Gitで変更履歴管理
  → 同じコードで同じ構成を何度でも作れる
  → レビューできる
```

### 実体験: LambdaのIAM修正
- AWSコンソールで直接変更ではなく、`iam.tf`を編集 → PR → apply
- これがIaCの作法

---

## セクション2: Terraformのワークフロー

### 基本の3ステップ
```
1. terraform init   → プロバイダやモジュールをダウンロード
2. terraform plan   → 何が変更されるかを確認
3. terraform apply  → 実際に変更を適用
```

### planの数字の読み方
```
0 to add     → 新しく作るリソース数
1 to change  → 変更するリソース数
0 to destroy → 削除するリソース数
```
**destroyが多いときは要注意！** → データベースやS3バケットが削除されるかも

### 実体験
```
iam.tfを編集（sqs:SendMessageを追加）
→ make plan-shared
  Plan: 0 to add, 1 to change, 0 to destroy
→ make apply-shared
  IAMポリシー更新完了
```

---

## セクション3: State（状態管理）

### stateとは
- Terraformが「今のインフラはこうなっている」と記録するファイル
- apply時は `state → 差分計算 → 必要な変更だけ実行` の流れ

### stateの保存場所
- ローカルだと危険（チームで共有できない）
- S3に保存するのが一般的
- DandoriAIは `dandori-terraform-state-bucket` を使用

### 環境ごとに分離
```
dandori-terraform-state-bucket/
├── poc/
├── production/
├── staging/
└── backend/
```

---

## セクション4: リソース定義の読み方

### 基本構文
```hcl
resource "リソースタイプ" "名前" {
  設定1 = 値1
  設定2 = 値2
}
```

### 5種類のブロック
```
resource  → 新しく作るリソース
data      → 既存のリソース情報を取得（読み取り専用）
variable  → 外部から渡す変数
output    → 他のモジュールに渡す値
locals    → ローカル変数（計算値）
```

### 実体験: ALBのidle_timeout変更
```hcl
resource "aws_lb" "poc" {
  name               = "${local.name_prefix}-alb"
  load_balancer_type = "application"
  subnets            = local.public_subnet_ids
  security_groups    = [aws_security_group.alb.id]
  idle_timeout       = 300  ← ここを追加
  tags               = local.tags
}
```

---

## セクション5: モジュール

### 再利用可能なコードの塊
```
dandori-infra/
├── modules/
│   └── poc-tenant/        ← 再利用可能な「テナント」モジュール
│       ├── ecs_app.tf
│       ├── ingress.tf
│       └── parameters.tf
│
└── environments/
    └── poc/
        └── tenants/
            └── main.tf    ← モジュールを使う側
```

### モジュールの使い方
```hcl
module "tenant_nhk_ep" {
  source      = "../../../modules/poc-tenant"
  tenant_key  = "nhk-ep"
  fqdn        = "nhk-ep.dandori.xincere.jp"
}

module "tenant_odakyu_ag" {
  source      = "../../../modules/poc-tenant"
  tenant_key  = "odakyu-ag"
  fqdn        = "odakyu-ag.dandori.xincere.jp"
}
```

### メリット
- テナント追加が簡単（モジュール呼び出しを1つ書くだけ）
- ECS・SQS・Lambda・S3・IAMが全部まとめて作られる

---

## セクション6: lifecycleブロック

### 主な指定方法
```hcl
lifecycle {
  ignore_changes = [value]    # 値の変更を無視
}

lifecycle {
  create_before_destroy = true  # 置き換え時に古いの消す前に新しいの作る
}

lifecycle {
  prevent_destroy = true        # terraform destroyを防ぐ
}
```

### 使い分け
- `ignore_changes`: SSMパラメータなど手動変更を保持したい値
- `create_before_destroy`: ダウンタイムを防ぎたいリソース
- `prevent_destroy`: 本番DBなど絶対に消したくないリソース

### 実体験: SSMパラメータのignore_changes
```
初回apply: 正しい値が入る
sync-ssm-params.sh: 手動変更される
その後のapply: ignore_changesで無視 → 手動変更が保持される
→ S3_BUCKETの設定ミスが放置される事態に
```

---

## セクション7: Terraform管理下のリソースはコンソールで触らない

### 手動変更が起きたらどうなるか
```
Terraformはstate（前回applyした状態）を記録している
→ AWSの実態を見る → stateと違う！
→ 次のterraform apply時にstateに戻そうとする
→ 手動変更が消える
```

### 例外: lifecycle { ignore_changes = [...] }
- これが設定されているフィールドだけは手動変更が保持される
- SSMパラメータがこのパターン

---

## 復習ポイント
- [ ] IaCのメリットを3つ挙げられるか？
- [ ] init, plan, applyの各コマンドの役割を説明できるか？
- [ ] planの「add, change, destroy」の読み方を説明できるか？
- [ ] stateとは何か、なぜS3に置くか説明できるか？
- [ ] resource, data, variable, outputの違いを説明できるか？
- [ ] モジュールを使うメリットを説明できるか？
- [ ] lifecycle ignore_changesがいつ役立つか説明できるか？
- [ ] 手動変更をすると何が起きるか説明できるか？
