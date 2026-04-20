# Day 10: SQS基礎 - キュー・メッセージ・DLQ

- 学習日: 2026-04-16
- 所要時間: 約25分

## セクション1: SQSとは

### 郵便ポストの例え
```
SQSがない場合:
  ECSが直接Lambdaに「この仕事やって」と頼む
  → Lambdaが忙しかったら失敗
  → ECSが死んだら依頼した仕事も消える

SQSがある場合:
  ECSが郵便ポスト（SQS）に手紙（メッセージ）を入れる
  → Lambdaが暇なときに取りに来る
  → ECSが死んでも手紙はポストに残る
```

### 役割
- 送る側と受ける側を**切り離す**（疎結合）
- 受ける側が処理しきれない量を**一時的にバッファリング**
- 片方が死んでも他方に影響しない

### 確認コマンド
```bash
# キュー一覧
aws sqs list-queues --queue-name-prefix dandori-poc-odakyu-ag

# キューの状態（メッセージ数など）
aws sqs get-queue-attributes --queue-url <url> \
  --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible \
  --query Attributes --output json
```

---

## セクション2: short/medium/longの使い分け

### なぜ3つに分けるか
- Lambdaのタイムアウトが用途ごとに異なる
- 重い処理が軽い処理をブロックしないようにする
- メモリやコストも処理ごとに最適化

### DandoriAIの構成
```
short:
  タイムアウト 60秒、メモリ 1024MB
  用途: メール送信、Slack通知

medium:
  タイムアウト 600秒（10分）、メモリ 1024MB
  用途: 学習データ取り込み

long:
  タイムアウト 900秒（15分）、メモリ 2048MB
  用途: ルールブック生成、AI見積
```

### 1つのキューだけだと何が問題か
- 全部長タイムアウトにするしかない → メール送信でも15分Lambda起動 → コスト無駄
- 重い処理がキューを占有して軽い処理が遅延する

---

## セクション3: メッセージのライフサイクル

### 4つの状態
```
1. 送信（SendMessage）
   ECSが「この仕事やって」とメッセージを投入
   → キューにメッセージが入る（Visible）

2. 受信（ReceiveMessage）
   Lambdaがメッセージを取り出す
   → 一時的に見えなくなる（NotVisible）
   → 他のLambdaが同じメッセージを取らないようにする

3-A. 成功 → 削除（DeleteMessage）
   処理完了 → メッセージをキューから削除

3-B. 失敗 → 戻る
   エラー or タイムアウト → メッセージが再びVisible
   → 別のLambdaが再処理（リトライ）
```

### 重要な属性
```
ApproximateNumberOfMessages:
  → Visibleなメッセージ数（処理待ち）

ApproximateNumberOfMessagesNotVisible:
  → NotVisibleなメッセージ数（処理中またはVisibility Timeout中）
```

---

## セクション4: Visibility Timeout

### 仕組み
```
Visibility Timeout = メッセージが「見えない」状態を維持する時間

Lambdaがメッセージを取得
→ Visibility Timeout（900秒）の間、他のLambdaからは見えない
→ 900秒以内に処理完了 → DeleteMessage → 成功
→ 900秒過ぎても完了しない → 再びVisible → リトライ
```

### 実体験: バッチ17で止まった原因
- 古いエラーメッセージが Visibility Timeout（15分）で隠れていた
- 15分経つまで再処理されなかった
- その間ずっと「処理中」に見えていた

### 確認コマンド
```bash
aws sqs get-queue-attributes \
  --queue-url <queue-url> \
  --attribute-names VisibilityTimeout \
  --query Attributes --output json
```

---

## セクション5: DLQ（デッドレターキュー）

### 役割
- 何度リトライしても失敗するメッセージの退避先
- 後で人間が確認して原因調査・修正できる

### リトライの流れ
```
通常キュー:
  受信 → 失敗 → リトライ1回目
  受信 → 失敗 → リトライ2回目
  ...
  受信 → 失敗 → リトライ5回目（maxReceiveCount=5）
  → もうダメ → DLQに移動
```

### 確認コマンド
```bash
# DLQのメッセージ数確認
aws sqs get-queue-attributes \
  --queue-url <dlq-url> \
  --attribute-names ApproximateNumberOfMessages \
  --query Attributes --output json
```

### ポイント
- DLQにメッセージが溜まっている → 何かが壊れている
- 通常の監視対象

---

## セクション6: SQSのパージ

### コマンド
```bash
aws sqs purge-queue --queue-url <queue-url>
```

### 注意点
- パージは最大60秒かかる
- パージ中に新しく投入されたメッセージも消える可能性がある
- 処理中のメッセージ（NotVisible）は消えない場合がある

### 実体験: パージを使った場面
- ルールブック生成を止めたいとき → longキューをパージ
- 古いリトライメッセージを消したいとき → パージ
- バッチ16→17で問題が起きた一因かもしれない

---

## 復習ポイント
- [ ] SQSの役割（疎結合）を説明できるか？
- [ ] short/medium/longに分ける理由を説明できるか？
- [ ] メッセージのライフサイクル（Visible/NotVisible/削除）を説明できるか？
- [ ] Visibility Timeoutの意味と、切れたら何が起きるか説明できるか？
- [ ] DLQの役割と、いつメッセージが移動するか説明できるか？
- [ ] 前回バッチ17で止まった問題をSQSの観点で説明できるか？
