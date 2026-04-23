# Hands-on Day 2: DynamoDB 基礎

- 実施日: 2026-04-24
- 所要時間: 約1時間
- SAA頻出度: ★★★（超頻出）

## 目的

DynamoDB（NoSQL）を体験し、RDSとの違いを理解する。
SAAでの「使い分けの判断問題」に答えられるレベルで仕組みを理解する。

## 作ったもの

- テーブル名: `todos-demo`
- パーティションキー: `userId`（String）
- ソートキー: `createdAt`（String）
- 3件のアイテム（ToDoリスト）

---

## DynamoDBの基本概念

### RDSとの違い

| | RDS (PostgreSQL) | DynamoDB |
|---|---|---|
| 種類 | リレーショナル | NoSQL（キーバリュー） |
| スキーマ | 固定（テーブル定義必須） | 柔軟（アイテムごと自由） |
| クエリ | 複雑なSQL、JOIN可能 | キーベースのみ、JOIN不可 |
| スケール | 縦スケール中心 | 横スケール、自動 |
| 一貫性 | 強い一貫性 | 結果整合性 or 強整合性 |
| パフォーマンス | 数十ms〜 | 1桁ms（超高速） |
| 得意 | 複雑な関係性のデータ | 大量データの高速アクセス |
| 苦手 | 超大量のリクエスト | 複雑な集計・JOIN |

### キーの設計

```
パーティションキー:
  データを振り分ける主キー
  → 同じuserIdのデータは同じパーティションに格納される
  → 検索が高速

ソートキー:
  パーティション内でソートする軸
  → 同じuserIdの中で createdAt でソート
  → 「このユーザーの最新のToDo」などが取れる
```

### スキーマレスの実例

```
アイテム1: { userId, createdAt, content, done }
アイテム2: { userId, createdAt, content, done, priority }
                                             ↑ アイテム1にはない属性
```

→ テーブル定義の変更なしに、新しい属性を追加できる

---

## 主要な操作（CLI）

### PutItem（追加）

```bash
aws dynamodb put-item \
  --table-name todos-demo \
  --item '{
    "userId": {"S": "user-001"},
    "createdAt": {"S": "2026-04-24T10:00:00"},
    "content": {"S": "牛乳を買う"},
    "done": {"BOOL": false}
  }' \
  --profile learning
```

### GetItem（1件取得、最速）

```bash
aws dynamodb get-item \
  --table-name todos-demo \
  --key '{
    "userId": {"S": "user-001"},
    "createdAt": {"S": "2026-04-24T10:00:00"}
  }' \
  --profile learning
```

### Query（パーティションキーで複数取得、高速）

```bash
aws dynamodb query \
  --table-name todos-demo \
  --key-condition-expression "userId = :uid" \
  --expression-attribute-values '{":uid": {"S": "user-001"}}' \
  --profile learning
```

### ソートキーで範囲指定

```bash
aws dynamodb query \
  --table-name todos-demo \
  --key-condition-expression "userId = :uid AND createdAt > :t" \
  --expression-attribute-values '{
    ":uid": {"S": "user-001"},
    ":t": {"S": "2026-04-24T10:30:00"}
  }' \
  --profile learning
```

### UpdateItem（一部属性だけ更新）

```bash
aws dynamodb update-item \
  --table-name todos-demo \
  --key '{...}' \
  --update-expression "SET done = :d" \
  --expression-attribute-values '{":d": {"BOOL": true}}' \
  --return-values ALL_NEW \
  --profile learning
```

### DeleteItem

```bash
aws dynamodb delete-item \
  --table-name todos-demo \
  --key '{...}' \
  --profile learning
```

### 主要操作まとめ

```
PutItem    → アイテム追加（新規 or 上書き）
GetItem    → パーティションキー + ソートキーで1件取得（最速）
Query      → パーティションキーを指定して複数取得（高速）
Scan       → テーブル全件スキャン（遅い、非推奨）
UpdateItem → 一部属性だけ更新
DeleteItem → 削除
```

---

## 型指定（DynamoDBの独特なJSON）

```
{"S": "..."}    文字列（String）
{"N": "123"}    数値（Number）※数値も文字列として渡す
{"BOOL": true}  ブール値
{"L": [...]}    リスト（配列）
{"M": {...}}    マップ（オブジェクト）
```

---

## SAA頻出パターン

### パターン1: 使い分け判断

```
DynamoDB向き:
- Eコマースのセッション管理
- IoTセンサーデータの高速書き込み
- ユーザープロファイル取得
- ゲームのリーダーボード
- ECサイトの商品カタログ（読み取り大量）

RDS向き:
- 注文と顧客と商品の複雑な関係性
- 月次売上レポート（集計）
- トランザクション多数
- 複雑なJOIN、分析クエリ
```

### パターン2: キャパシティモード

```
オンデマンド（On-Demand）:
  - 使った分だけ課金
  - トラフィック予測困難な場合に最適
  - 例: DandoriAIのような不定期アクセス

プロビジョンド（Provisioned）:
  - 事前に読み書きの上限を予約
  - 安定トラフィック向き、安い
  - Auto Scaling併用可
```

### パターン3: 高可用性・パフォーマンス機能

```
DynamoDB Streams:
  アイテムの変更をリアルタイム検知
  → Lambdaトリガーで「データ追加時に○○する」を実現

Global Tables:
  マルチリージョン同期
  → 世界中のユーザーに低レイテンシ
  → SAAの「グローバル配信」問題で頻出

DAX（DynamoDB Accelerator）:
  DynamoDB専用のキャッシュ
  → マイクロ秒レベルのアクセス
```

---

## 畑の例えで振り返り

```
RDS（今まで学んだ）:
  倉庫の中にきっちり設計された棚
  「projectsテーブル」には必ず id、name、status の欄がある
  欄が足りなければ「この倉庫を改修」する必要がある

DynamoDB:
  倉庫の中に自由な箱が並んでいる
  箱にラベル（パーティションキー）が貼ってあり
  ラベルでグループ化されている
  箱の中身（属性）は箱ごとに違ってもOK
```

---

## 学んだこと（SAA対策の重要ポイント）

- [ ] RDSとDynamoDBの使い分けを判断できる
- [ ] パーティションキーとソートキーの役割を説明できる
- [ ] スキーマレスの意味とメリットを説明できる
- [ ] PutItem / GetItem / Query / Scan の違いを説明できる
- [ ] Scanが推奨されない理由を説明できる（全件スキャンで遅い・高コスト）
- [ ] オンデマンド vs プロビジョンドの使い分けを説明できる
- [ ] Global Tables / DynamoDB Streams / DAX の用途を説明できる

---

## 後片付け

データがあっても月5GBまで無料。放置してもOK。

削除する場合:
```bash
aws dynamodb delete-table --table-name todos-demo --profile learning
```

---

## 次回のハンズオン予定

Day 3: Auto Scaling（EC2 Auto Scaling または DynamoDB Auto Scaling）
