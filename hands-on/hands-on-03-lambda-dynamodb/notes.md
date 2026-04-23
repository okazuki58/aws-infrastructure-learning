# Hands-on Day 3: Lambda + DynamoDB でサーバーレスAPI

- 実施日: 2026-04-24
- 所要時間: 約1時間
- SAA頻出度: ★★★（超頻出）

## 目的

Lambda関数を作成し、DynamoDBと連携してサーバーレスAPIを構築する
Function URLを使ってAPIを公開する

## 作ったもの

```
ブラウザ/curl
  ↓ HTTPS
Function URL（Lambdaの公開エンドポイント）
  ↓
Lambda関数（Node.js）
  ↓
DynamoDB（todos-demo）
```

---

## セットアップ手順

### Step 1: IAMロール作成

Lambda用のロール（DynamoDBアクセス + CloudWatch Logs）:
- ロール名: `lambda-todos-demo-role`
- 許可ポリシー:
  - `AmazonDynamoDBReadOnlyAccess`
  - `AWSLambdaBasicExecutionRole`

### Step 2: Lambda関数作成

- 関数名: `todos-demo-api`
- ランタイム: Node.js 22.x
- 既存のロールを使用: `lambda-todos-demo-role`
- 関数URLを有効化（認証タイプ: NONE）

### Step 3: DynamoDBテーブル再作成

```bash
aws dynamodb create-table \
  --table-name todos-demo \
  --attribute-definitions \
    AttributeName=userId,AttributeType=S \
    AttributeName=createdAt,AttributeType=S \
  --key-schema \
    AttributeName=userId,KeyType=HASH \
    AttributeName=createdAt,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --profile learning
```

PAY_PER_REQUEST = オンデマンドモード（使った分だけ課金）

### Step 4: テストデータ投入（3件）

user-001に2件、user-002に1件

### Step 5: Lambdaコード（改善版）

```javascript
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, QueryCommand } from "@aws-sdk/lib-dynamodb";

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

export const handler = async (event) => {
  try {
    const userId = event.queryStringParameters?.userId;

    if (!userId) {
      return {
        statusCode: 400,
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ error: "userId query parameter is required" }),
      };
    }

    const command = new QueryCommand({
      TableName: "todos-demo",
      KeyConditionExpression: "userId = :uid",
      ExpressionAttributeValues: {
        ":uid": userId,
      },
    });

    const response = await docClient.send(command);

    return {
      statusCode: 200,
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        userId,
        count: response.Count,
        items: response.Items,
      }),
    };
  } catch (error) {
    return {
      statusCode: 500,
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ error: error.message }),
    };
  }
};
```

### 動作確認

```bash
# 正しいuserId → 2件返る
curl "https://xxxxx.lambda-url.ap-northeast-1.on.aws/?userId=user-001"

# 存在しないuserId → 0件
curl "https://xxxxx.lambda-url.ap-northeast-1.on.aws/?userId=001"

# パラメータなし → 400エラー
curl "https://xxxxx.lambda-url.ap-northeast-1.on.aws/?user=001"
```

---

## 学んだこと

### Function URL

Lambdaに直接URLを付ける機能。API Gatewayを使わずにLambdaを公開できる。

- メリット: シンプル、安い、即デプロイ
- デメリット: API Gatewayほど高度な機能はない（認証、レート制限、変換など）

### Scan vs Query

```
Scan:
  テーブル全件をスキャン
  → 100万件あれば100万件分スキャン
  → 遅い、高い
  → 本番非推奨

Query:
  パーティションキーで絞り込み
  → 該当パーティションのデータだけ取得
  → 速い、安い
  → 本番推奨
```

### サーバーレスAPIのメリット

```
1. サーバーを1台も立てない
   - EC2なし、ECSなし
   - インフラ管理不要

2. 常時稼働しない
   - リクエストが来た瞬間だけ起動
   - アクセスがないと料金ゼロ

3. 自動スケール
   - 1リクエストでも1万リクエストでも動く
   - Lambdaが自動で並列起動
   - DynamoDBもオンデマンドで自動スケール
```

### SDK の使い方

```javascript
// 1. クライアント作成（引数なし → 自動認証）
const client = new DynamoDBClient({});

// 2. DocumentClient でラップ（JSONをそのまま扱える）
const docClient = DynamoDBDocumentClient.from(client);

// 3. コマンド作成 → 送信
const command = new QueryCommand({ ... });
await docClient.send(command);
```

---

## SAA頻出パターン

### パターン1: サーバーレスAPI構築
```
問題: 低頻度だが突発的なアクセスがあるAPIを安く構築したい
回答: Lambda + DynamoDB
理由:
  - 使った分だけ課金（アイドル時ゼロ円）
  - Lambda自動スケール
  - DynamoDBもオンデマンド
  - サーバー管理不要
```

### パターン2: 超高速応答が必要
```
問題: ユーザープロファイル取得APIで数ミリ秒の応答が必要
回答: API Gateway + Lambda + DynamoDB（+ DAX）
理由:
  - DynamoDBは1桁ms応答
  - DAXでキャッシュ追加でマイクロ秒
```

### パターン3: Function URL vs API Gateway
```
Function URL:
  シンプルな公開エンドポイント
  認証不要、または簡易な場合
  低コスト

API Gateway:
  認証（Cognito連携）
  レート制限
  リクエスト/レスポンス変換
  ステージング環境
  複雑なAPI管理が必要な場合
```

---

## 畑の例えで振り返り

```
Lambda（出張シェフ）:
  普段いない、呼ばれたら来る
  呼び出し方:
    - SQSから呼ばれる（非同期、DandoriAIのパターン）
    - Function URL（直接呼ばれる、今回のパターン）
    - API Gateway経由（もっと高機能）

DynamoDB:
  自由形式の箱倉庫
  箱のラベル（パーティションキー）で即取り出せる
```

---

## 学んだことのチェックリスト

- [ ] Lambda関数の作成手順
- [ ] IAMロールを関数に割り当てる方法
- [ ] Function URLの仕組み
- [ ] event.queryStringParameters でパラメータを取得する方法
- [ ] Scan と Query の違い
- [ ] DynamoDBClient と DocumentClient の使い方
- [ ] サーバーレスAPIのメリットとコスト特性
- [ ] Function URL と API Gateway の使い分け

---

## 後片付け

Lambda + Function URL + DynamoDBは呼ばれなければほぼ課金されない。
気になる場合:

```bash
# Lambda関数削除
aws lambda delete-function --function-name todos-demo-api --profile learning

# DynamoDB削除
aws dynamodb delete-table --table-name todos-demo --profile learning

# IAMロール削除（先にポリシーをデタッチ）
aws iam detach-role-policy \
  --role-name lambda-todos-demo-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonDynamoDBReadOnlyAccess \
  --profile learning

aws iam detach-role-policy \
  --role-name lambda-todos-demo-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole \
  --profile learning

aws iam delete-role --role-name lambda-todos-demo-role --profile learning
```

---

## 次回予定

Day 4: API Gateway + Lambda + DynamoDB（今回のFunction URLをAPI Gatewayに置き換え）
または
Day 4: Auto Scaling体験
