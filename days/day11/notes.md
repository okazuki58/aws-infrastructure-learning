# Day 11: Lambda基礎 - 関数・トリガー・環境変数

- 学習日: 2026-04-16
- 所要時間: 約30分

## セクション1: Lambdaとは

### 関数を実行するサービス
- 関数1つだけを登録しておいて、呼ばれたときだけ実行される
- サーバーを常時動かさない（サーバーレス）
- 実行時間分だけ課金、アイドル時間はゼロ円

### ECSとLambdaの違い
```
ECS: レストランのキッチン
  → 営業時間中ずっとコック（コンテナ）がいる
  → 注文（リクエスト）が来たらすぐ調理
  → 注文がなくてもコックは店にいる（給料発生）

Lambda: 出張シェフ
  → 普段は誰もいない
  → 注文が入ったとき呼ばれて来る
  → 料理を作って帰る
  → 料理を作った時間分だけ料金
```

### DandoriAIの使い分け
```
常時必要な処理（Webアプリ） → ECS
  → Next.jsアプリ、API全般

不定期な重い処理 → Lambda経由（SQS）
  → AI見積生成、ルールブック生成
  → 学習データ取り込み、メール送信
```

### コストの考え方
- ECS: タスク1個24時間稼働で月額約30ドル
- Lambda: 1回15分×1日10回で月額約5ドル
- 重い処理をLambdaに切り出すことでECSのAuto Scalingを抑えられる

---

## セクション2: Lambdaの制約

### 主な制約
```
- 最大実行時間: 15分（900秒）
- 最大メモリ: 10,240MB
- デプロイパッケージサイズ: 250MB（zip解凍後）
- 一時ディスク: 512MB〜10,240MB
- 同時実行数: 1,000（デフォルト）
```

### 実体験: 15分の壁
```
ルールブック生成: 34バッチ × 40秒 = 約23分
→ 15分の壁を超えてタイムアウト

対策:
案1（ダメだった）: 1バッチごとにSQSに再投入
  → チェーンが34回 → 17バッチ目で止まる

案2（安定した）: 10分ごとにSQSに再投入
  → チェーンが2-3回 → 安定動作
```

15分を超える処理は**分割してSQSで繋ぐ**のが定石。

### DandoriAIのLambda構成
```
short:  60秒, 1024MB   ← 軽い処理（メール、Slack通知）
medium: 600秒, 1024MB  ← 中程度（学習データ取り込み）
long:   900秒, 2048MB  ← 重い処理（ルールブック、AI見積）
```

### 確認コマンド
```bash
aws lambda list-functions \
  --query "Functions[?contains(FunctionName, 'dandori-poc-odakyu-ag')].{Name:FunctionName,Runtime:Runtime,Memory:MemorySize,Timeout:Timeout}" \
  --output table
```

---

## セクション3: Lambdaトリガー

### 主なトリガー
```
SQS: キューにメッセージが入ったら起動 ← DandoriAIで使用
S3: バケットにファイルが置かれたら起動
EventBridge: 定期実行（cronのような）
API Gateway: HTTPリクエストで起動
DynamoDB Streams: DBの変更で起動
```

### SQSトリガーの動作
```
1. SQSキューにメッセージが入る
2. SQSイベントソースマッピングが検知
3. Lambdaを起動し、メッセージを渡す
4. Lambda処理完了でメッセージ削除
```

### 設定確認
```bash
aws lambda list-event-source-mappings \
  --function-name dandori-poc-odakyu-ag-lambda-sqs-processor-long \
  --query "EventSourceMappings[*].{State:State,BatchSize:BatchSize,Source:EventSourceArn}" \
  --output json
```

### 重要な属性
- `State: Enabled` → トリガーが有効
- `BatchSize: 5` → 1回の起動で最大5件のメッセージを処理

---

## セクション4: Lambdaの環境変数

### ECSとLambdaの違い（復習）
```
ECS:
  タスク定義にSSM ARNを記載
  → 起動時にECSがSSMから値を取得してコンテナの環境変数にセット

Lambda:
  TerraformがSSMの値を読み取ってLambdaの環境変数に直接書き込む
  → terraform apply時に書き込まれる
  → SSM変更だけでは反映されない
```

### 実体験
```
1. 「AWS_ACCOUNT_ID environment variable is required」
   → Lambda環境変数にAWS_ACCOUNT_IDがなかった
   → Terraformで追加 → terraform apply

2. 「sqs:sendmessage ... no identity-based policy allows」
   → LambdaロールにSendMessage権限がなかった
   → iam.tfに追加 → terraform apply
```

どちらもTerraform経由での修正が必要。

### 確認コマンド
```bash
aws lambda get-function-configuration \
  --function-name <function-name> \
  --query "Environment.Variables" --output json
```

---

## セクション5: Lambdaランタイムの制約

### 動作環境
- 純粋なNode.js 22環境
- フル機能のLinuxではない

### 使えないもの
- ブラウザDOM API（DOMMatrix, Path2D, canvas等）
- Puppeteer / Chromium（含まれていない）
- pdfjs-distなどブラウザAPI依存のライブラリ

### 使えるもの
- 純粋なNode.jsライブラリ
- @prisma/client（外部として指定済み）
- pg（PostgreSQLクライアント）
- 軽量なHTTPライブラリ

### 注意点
- ECSでは動くコードがLambdaでは動かないことがある
- ECS: フル機能のLinux環境
- Lambda: 限定された実行環境
- Lambda側で呼ばれるコードに新しい依存を追加するときは要注意

---

## Auto Scalingとの関係

### 重い処理をECSだけでやると
```
10人が同時にルールブック生成
→ ECS負荷急上昇 → Auto Scalingでタスク増加
→ 15分間、複数台分のFargate料金
→ 通常リクエストも処理しないといけないのでタスク不足の可能性
```

### Lambdaに切り出すと
```
10人が同時にルールブック生成
→ ECSは「受け付けました」を返すだけ
→ Lambdaが10個並列で起動（自動）
→ Lambdaは使った分だけ課金
→ ECSのタスク数は増やさなくていい
```

---

## 復習ポイント
- [ ] LambdaとECSの違いを説明できるか？
- [ ] Lambdaの15分制限と、超える場合の対策を説明できるか？
- [ ] SQSトリガーの動作を説明できるか？
- [ ] Lambdaの環境変数とSSMの関係を説明できるか？
- [ ] Lambdaで使えないライブラリの特徴を説明できるか？
- [ ] なぜECSとLambdaを使い分けるか（コストとスケーリング）を説明できるか？
