# Hands-on Day 4: API Gateway + Lambda + DynamoDB

- 実施日: 2026-04-24
- 所要時間: 約1時間
- SAA頻出度: ★★★（超頻出）

## 目的

Function URL を API Gateway に置き換え、REST API を構築する
GET（一覧取得）、POST（新規作成）、PUT（更新）の3つのエンドポイントを実装

## 作ったもの

```
ブラウザ/curl
  ↓ HTTPS
API Gateway（HTTP API、東京リージョン）
  ├─ GET /todos                        → 一覧取得
  ├─ POST /todos                       → 新規作成
  └─ PUT /todos/{userId}/{createdAt}  → 更新
    ↓
Lambda関数（1つで3つのエンドポイントを処理）
  ↓
DynamoDB（todos-demo）
```

---

## Function URL vs API Gateway

| | Function URL | API Gateway (HTTP API) |
|---|---|---|
| セットアップ | 超簡単 | 少し手間 |
| 認証 | IAMのみ | Cognito/JWT/IAMなど豊富 |
| レート制限 | なし | あり |
| カスタムドメイン | 不可 | 可能 |
| REST API風設計 | 難しい | 簡単 |
| ステージング | なし | あり（dev/prod） |
| コスト | 安い | 少し高い（HTTP APIは70%安い） |

**使い分け**:
- Function URL: シンプルな1エンドポイント、内部ツール
- API Gateway: 公開API、認証必要、REST風設計

---

## REST API vs HTTP API

```
REST API（API Gateway の古い方）:
  高機能、古くからある
  VPC Links、ワークフロー対応
  高コスト

HTTP API（新しい方、今回使用）:
  70%安い
  シンプル、必要な機能は揃う
  SAAで「安くサーバーレス API」はこれが正解
```

---

## 作成手順

### Step 1: Lambdaコードの拡張

GET / POST / PUT の3つを1つのLambdaでハンドリング。

```javascript
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import {
  DynamoDBDocumentClient,
  QueryCommand,
  PutCommand,
  UpdateCommand,
} from "@aws-sdk/lib-dynamodb";

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);
const TABLE_NAME = "todos-demo";

const jsonResponse = (statusCode, body) => ({
  statusCode,
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(body),
});

export const handler = async (event) => {
  try {
    const method = event.requestContext?.http?.method || event.httpMethod;
    const path = event.rawPath || event.path;

    if (method === "GET" && path === "/todos") return await getTodos(event);
    if (method === "POST" && path === "/todos") return await createTodo(event);
    if (method === "PUT" && path.startsWith("/todos/")) return await updateTodo(event);

    return jsonResponse(404, { error: "Not Found" });
  } catch (error) {
    return jsonResponse(500, { error: error.message });
  }
};

async function getTodos(event) {
  const userId = event.queryStringParameters?.userId;
  if (!userId) return jsonResponse(400, { error: "userId required" });

  const response = await docClient.send(
    new QueryCommand({
      TableName: TABLE_NAME,
      KeyConditionExpression: "userId = :uid",
      ExpressionAttributeValues: { ":uid": userId },
    })
  );

  return jsonResponse(200, {
    userId,
    count: response.Count,
    items: response.Items,
  });
}

async function createTodo(event) {
  const body = JSON.parse(event.body || "{}");
  const { userId, content, priority } = body;

  if (!userId || !content) {
    return jsonResponse(400, { error: "userId and content required" });
  }

  const item = {
    userId,
    createdAt: new Date().toISOString(),
    content,
    done: false,
  };
  if (priority !== undefined) item.priority = priority;

  await docClient.send(new PutCommand({ TableName: TABLE_NAME, Item: item }));
  return jsonResponse(201, { message: "Created", item });
}

async function updateTodo(event) {
  const parts = (event.rawPath || event.path).split("/");
  const userId = decodeURIComponent(parts[2]);
  const createdAt = decodeURIComponent(parts[3]);

  const body = JSON.parse(event.body || "{}");
  const { done } = body;

  if (done === undefined) return jsonResponse(400, { error: "done required" });

  const response = await docClient.send(
    new UpdateCommand({
      TableName: TABLE_NAME,
      Key: { userId, createdAt },
      UpdateExpression: "SET done = :d",
      ExpressionAttributeValues: { ":d": done },
      ReturnValues: "ALL_NEW",
    })
  );

  return jsonResponse(200, { message: "Updated", item: response.Attributes });
}
```

### Step 2: Lambda権限追加

`AmazonDynamoDBFullAccess` ポリシーを追加（書き込み権限）

### Step 3: API Gateway 作成

1. HTTP API を選択（REST APIではない）
2. 統合: Lambda → todos-demo-api
3. ルート3つ作成:
   - GET /todos
   - POST /todos
   - PUT /todos/{userId}/{createdAt}
4. ステージ: $default、自動デプロイON

**注意**: Lambda と API Gateway は必ず同じリージョン（東京）で作る！

---

## 動作確認

### GET
```bash
curl "https://xxxxx.execute-api.ap-northeast-1.amazonaws.com/todos?userId=user-001"
# → count と items のJSON
```

### POST
```bash
curl -X POST "https://xxxxx.execute-api.ap-northeast-1.amazonaws.com/todos" \
  -H "Content-Type: application/json" \
  -d '{"userId": "user-001", "content": "新しいToDo", "priority": 3}'
# → {"message": "Created", "item": {...}}
```

### PUT（URLエンコードに注意）
```bash
# createdAt: 2026-04-23T16:32:04.555Z → 2026-04-23T16%3A32%3A04.555Z
curl -X PUT "https://xxxxx.execute-api.ap-northeast-1.amazonaws.com/todos/user-001/2026-04-23T16%3A32%3A04.555Z" \
  -H "Content-Type: application/json" \
  -d '{"done": true}'
# → {"message": "Updated", "item": {..., "done": true}}
```

---

## 学んだこと

### API Gateway の役割
- APIの「玄関」
- ルーティング（メソッド + パス → Lambda）
- 認証、レート制限、変換などの処理をLambdaの手前で実行
- Lambdaは純粋なビジネスロジックに集中できる

### 1 Lambda + 複数エンドポイント
- `event.requestContext.http.method` と `event.rawPath` でルーティング
- 関数が1つで済むのでコスト安、管理が楽
- 関数を分ける場合は「単一責任の原則」で分離（設計判断）

### パスパラメータ
- `{userId}` のような動的な部分
- event.pathParameters または event.rawPath から取得
- URLエンコードに注意（`:` → `%3A`）

### リージョンの重要性
- Lambda、DynamoDB、API Gatewayは同じリージョンにあるべき
- 別リージョンだと遅延が大きい、設定が面倒

---

## SAA頻出パターン

### パターン1: サーバーレスAPI
```
問題: ECサイトのユーザーAPIを作りたい。
      アクセスが不定期、スケーラブルで低コストにしたい。
回答: API Gateway (HTTP API) + Lambda + DynamoDB
理由:
  - アイドル時ゼロ円
  - 自動スケール
  - サーバー管理不要
```

### パターン2: API Gatewayの選択
```
問題: サーバーレスAPIを安く構築したい
回答: HTTP API（REST APIより70%安い）

問題: WebSocketが必要
回答: WebSocket API

問題: プライベートAPIをVPC内で使いたい
回答: REST API（VPC Link対応）、HTTP APIは非対応
```

### パターン3: 認証
```
問題: Cognitoユーザーを認証してAPIに接続
回答: API Gateway + Cognito オーソライザー

問題: 別のLambdaでカスタム認証したい
回答: API Gateway + Lambda オーソライザー

問題: IAM認証したい
回答: API Gateway + IAM認証
```

---

## 畑の例えで振り返り

```
今までの畑（DandoriAI）:
  ALB（受付）→ ECS工場 → RDS倉庫
  → 常時稼働するWebアプリ

今回作ったもの:
  API Gateway（小さい受付）→ Lambda（出張シェフ）→ DynamoDB（箱倉庫）
  → リクエストがあるときだけ動く、サーバーレスAPI
```

---

## 後片付け

全てサーバーレスなので、呼ばれなければほぼ課金されない。
気になる場合:

```bash
# Lambda関数削除
aws lambda delete-function --function-name todos-demo-api --profile learning

# API Gateway削除（コンソールから）
# API Gateway → todos-demo-http-api → 削除

# DynamoDB削除
aws dynamodb delete-table --table-name todos-demo --profile learning
```

---

## 学んだことチェックリスト

- [ ] REST API と HTTP API の違いを説明できる
- [ ] Function URL と API Gateway の使い分けを説明できる
- [ ] 1つのLambdaで複数エンドポイントをハンドリングする方法を理解している
- [ ] パスパラメータとクエリパラメータの取得方法を理解している
- [ ] サーバーレスAPIの構成要素（API Gateway + Lambda + DynamoDB）を即答できる
- [ ] SAAで出される「サーバーレスAPI」問題に対応できる

---

## 次回予定

Day 5: VPC Endpoint または Auto Scaling
