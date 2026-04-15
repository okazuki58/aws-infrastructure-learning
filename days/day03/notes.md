# Day 3: ネットワーク実践 - DandoriAIのVPC構成を読み解く

- 学習日: 2026-04-05
- 所要時間: 約20分
- きっかけ: Day 1-2の知識を統合し、実際のTerraformコードと紐付ける

## セクション1: ALBのリスナールールによるテナント振り分け

### 学んだこと
- POC環境では複数テナント（nhk-ep, odakyu-ag）が**同じALB**を共有
- `dig`で確認すると、両テナントとも同じIPアドレスを返す
- ALBは**HTTPリクエストのHostヘッダー**を見てテナントを判別

### 振り分けの仕組み
```
ブラウザが https://odakyu-ag.dandori.xincere.jp にアクセス
  → DNS: 13.115.34.44（ALBのIP）を返す
  → ALB: Hostヘッダー "odakyu-ag.dandori.xincere.jp" を確認
    → リスナールール: odakyu-agのECSに転送
```

### Terraformコードの読み方
```hcl
# ingress.tf（modules/poc-tenant/）
resource "aws_lb_listener_rule" "production_fqdn_https_forward" {
  listener_arn = var.alb_listener_arn

  condition {
    host_header {
      values = [var.fqdn]  # "odakyu-ag.dandori.xincere.jp"
    }
  }

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn  # このテナントのECSに転送
  }
}
```

### つまづいたところ
- 最初「サブドメインだからRoute53で振り分けている」と思った
- → Route53はIPを返すだけ。振り分けはALBのリスナールールの仕事

---

## セクション2: Day 1-3の知識統合

### リクエストの全体フロー（アップロードの例）
```
1. ブラウザ → DNS（Route53）→ ALBのIP取得

2. ブラウザ → インターネットゲートウェイ → ALB（パブリックサブネット）
   - セキュリティグループ: ポート443、世界中からOK

3. ALB → Hostヘッダーで判定 → テナントのECSに転送
   - セキュリティグループ: ポート3000、ALBからのみ

4. ECS（プライベートサブネット）でNext.jsがリクエスト処理
   - busboyでZIPをストリーム解析

5. ECS → S3にファイルアップロード
   - IAM: タスクロールにs3:PutObject権限あり
```

### 2層のセキュリティ
| レイヤー | 仕組み | 例 |
|---------|--------|-----|
| ネットワーク | セキュリティグループ | ECSはALBからのポート3000のみ許可 |
| 権限 | IAM | タスクロールにS3/SQS権限を付与 |

---

## 実体験との紐付け
- `dig`コマンド: nhk-epとodakyu-agが同じIPを返す → ALBで振り分けていることを確認
- SSMのS3_BUCKET修正: テナントごとに異なるバケットを参照する設計
- busboyストリーミング修正: ECS内でのリクエスト処理（Step 4に該当）
- Terraformコード: `modules/poc-tenant/ingress.tf`のリスナールールを実際に読んだ

---

## 復習ポイント
- [ ] 同じALBで複数テナントを振り分ける仕組みを説明できるか？
- [ ] Route53とALBの役割の違いを説明できるか？
- [ ] ファイルアップロードの全体フロー（DNS → ALB → ECS → S3）を説明できるか？
- [ ] セキュリティグループとIAMの2層構造を説明できるか？
- [ ] `aws_lb_listener_rule`のTerraformコードを読めるか？
