# Day 5: ALB基礎 - ターゲットグループ・リスナー・ヘルスチェック

- 学習日: 2026-04-16
- 所要時間: 約25分

## セクション1: ALBの3つの構成要素

### レストランの例え
```
ALB = レストランの受付

リスナー = 受付の窓口
  ポート443（HTTPS）の窓口、ポート80（HTTP）の窓口

リスナールール = 受付マニュアル
  「予約名が○○のお客様は2階へ」（Hostヘッダーで判断）

ターゲットグループ = 各フロアのスタッフチーム
  odakyu-agのECS、nhk-epのECS など
```

### DandoriAI POCの構成
```
ALB（dandori-poc-alb）
├── リスナー: ポート80（HTTP）→ 404を返す
├── リスナー: ポート443（HTTPS）
│   ├── ルール: Host=odakyu-ag.dandori.xincere.jp → odakyu-ag-tg
│   ├── ルール: Host=nhk-ep.dandori.xincere.jp → nhk-ep-tg
│   └── デフォルト: fixed-response
├── ターゲットグループ: dandori-poc-odakyu-ag-tg（ポート3000）
├── ターゲットグループ: dandori-poc-nhk-ep-tg（ポート3000）
└── ターゲットグループ: dandori-poc-demo-tg（ポート3000）
```

### 確認コマンド
```bash
# ターゲットグループ一覧
aws elbv2 describe-target-groups --query "TargetGroups[?contains(TargetGroupName, 'poc')].{Name:TargetGroupName,Port:Port,HealthCheck:HealthCheckPath}" --output table

# リスナー一覧
aws elbv2 describe-listeners --load-balancer-arn <alb-arn> --query "Listeners[*].{Port:Port,Protocol:Protocol,Action:DefaultActions[0].Type}" --output table
```

---

## セクション2: ヘルスチェック

### なぜ必要か
```
ヘルスチェックがない場合:
  ECSタスクが死んでいるのにALBがリクエストを送り続ける → 502エラー

ヘルスチェックがある場合:
  定期的に /api/health にリクエスト → 失敗 → 異常と判定
  → そのタスクへの転送を停止 → ECSが新しいタスクを起動 → 復旧
```

### 設定値の意味
```
Path: /api/health     → チェック先のURL
Interval: 30          → 30秒ごとにチェック
Timeout: 5            → 5秒以内に返事がなければ失敗
Matcher: 200-399      → このステータスコードなら「元気」
Healthy: 2            → 2回連続成功で「正常」判定
Unhealthy: 2          → 2回連続失敗で「異常」判定
```

### 異常判定までの最大時間
- 30秒間隔 × 2回失敗 = 最大約60秒
- この間ユーザーは502エラーを見続ける可能性がある

### 実体験
- OOM Killでタスクが死ぬ → ヘルスチェック失敗 → 502 → 新タスク起動 → 復旧
- この一連の流れで数分間サービスが不安定だった

---

## セクション3: idle_timeout

### ヘルスチェックTimeoutとの違い
```
ヘルスチェックTimeout（5秒）:
  ALB → ECS に「元気？」と聞いて、5秒以内に返事がなければ「異常かも」
  → タスクの生死を判断するためのもの

idle_timeout（300秒）:
  ブラウザ ←→ ALB ←→ ECS の通信で、300秒間データが流れなければ接続を切る
  → 使われていない接続を片付けるためのもの
```

### DandoriAIでの設定
- デフォルト: 60秒
- 変更後: 300秒
- 理由: SSEストリーミング（AI処理の進捗配信）が最大10分間続くため

---

## セクション4: リスナーとポート

### ポート80とポート443の役割
```
ポート80  = HTTP（暗号化なし） → http://example.com
ポート443 = HTTPS（暗号化あり）→ https://example.com
```

### なぜ両方あるか
- ユーザーがhttp://でアクセスしてくる場合がある
- production: ポート80 → HTTPSにリダイレクト（301）
- POC: ポート80 → 404を返す
- 実際の処理は全てポート443（HTTPS）で行う

---

## 復習ポイント
- [ ] ALBの3つの構成要素（リスナー・リスナールール・ターゲットグループ）を説明できるか？
- [ ] ヘルスチェックの各設定値の意味を説明できるか？
- [ ] ヘルスチェックTimeoutとidle_timeoutの違いを説明できるか？
- [ ] ポート80とポート443の役割の違いを説明できるか？
- [ ] ECSタスクが死んでから復旧するまでの流れを説明できるか？
