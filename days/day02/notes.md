# Day 2: セキュリティグループとIAM - 誰が何にアクセスできるか

- 学習日: 2026-04-05
- 所要時間: 約30分
- きっかけ: LambdaからSQSへのSendMessageがAccessDeniedで失敗 → IAMの理解が必要と実感

## セクション1: セキュリティグループ

### 学んだこと
- **セキュリティグループ** = サブネットの各リソースに付ける「ドア」
- **インバウンドルール**: 入ってくる通信を許可
- **アウトバウンドルール**: 出ていく通信を許可
- ソースにはIPアドレスだけでなく、**セキュリティグループID**も指定できる

### DandoriAI POCのセキュリティグループ
| SG名 | 用途 |
|------|------|
| sg-alb | ALB（受付）|
| sg-ecs | ECSコンテナ |
| sg-lambda | Lambda |
| sg-weaviate | Weaviate（ベクトルDB）|
| sg-efs | EFS（共有ストレージ）|
| sg-bastion | 踏み台サーバー |

### ALBのインバウンドルール
```
ポート80 (HTTP)  + ポート443 (HTTPS) → 0.0.0.0/0（世界中からOK）
```

### ECSのインバウンドルール
```
ポート3000 → sg-alb（ALBからのみOK）
```

### ポイント
- ALBは誰でも入れる。ECSはALBからしか入れない
- これが「プライベートサブネットに置く意味」

### 確認コマンド
```bash
# セキュリティグループ一覧
aws ec2 describe-security-groups --filters "Name=vpc-id,Values=vpc-0dbdb809c78123ead" --query "SecurityGroups[*].{Name:GroupName,ID:GroupId}" --output table

# 特定SGのインバウンドルール
aws ec2 describe-security-groups --group-ids <sg-id> --query "SecurityGroups[0].IpPermissions[*]" --output json
```

---

## セクション2: IAM（Identity and Access Management）

### 学んだこと
- **セキュリティグループ** = 建物のドア（通信が届くか）
- **IAM** = 社員証と権限カード（操作できるか）
- ネットワーク的に到達できても、IAM権限がなければ操作不可

### IAMの3つの概念
| 概念 | 意味 | 例 |
|------|------|-----|
| ユーザー | 人間のアカウント | AWSコンソールログイン |
| ロール | サービスに割り当てる「社員証」 | ECSタスクロール |
| ポリシー | 「何ができるか」の権限リスト | S3読み書きOK |

### ポリシーのJSON構造
```json
{
  "Effect": "Allow",           // 許可 or 拒否
  "Action": ["sqs:SendMessage"], // 何ができるか
  "Resource": "arn:aws:sqs:...:dandori-poc-*"  // どのリソースに対して
}
```

### 最小権限の原則
- 必要な権限だけを与える
- ECSとLambdaで別々のロール → 万が一の被害を最小化
- 今日の体験: ECSにはsqs:SendMessageがあったのにLambdaにはなかった

---

## セクション3: ロールの仕組み

### 信頼ポリシーと権限ポリシー
```
ロール（社員証）
├── 信頼ポリシー: 誰がこのロールを使えるか
│   例: "ecs-tasks.amazonaws.com" → ECSだけが使える
│   例: "lambda.amazonaws.com" → Lambdaだけが使える
│
└── 権限ポリシー: 何ができるか
    ├── S3の読み書き
    ├── SQSの受信・送信
    └── SSMの読み取り
```

### 信頼ポリシーの読み方
```json
{
  "Effect": "Allow",                    // 許可する
  "Principal": {
    "Service": "lambda.amazonaws.com"   // Lambdaサービスが
  },
  "Action": "sts:AssumeRole"           // このロールを使うことを
}
```

### 確認コマンド
```bash
# 信頼ポリシーの確認
aws iam get-role --role-name dandori-poc-task-role --query "Role.AssumeRolePolicyDocument" --output json

# 権限ポリシーの確認
aws iam get-role-policy --role-name dandori-poc-lambda-sqs-processor-role --policy-name <policy-name> --output json
```

---

## セクション4: ECSのロールは2種類

### タスク実行ロール vs タスクロール
```
ECS:
  ECSサービス（起動する人）→ タスク実行ロール
    - ECRからイメージpull
    - SSMからパラメータ読み取り
    - CloudWatch Logsにログ出力

  コンテナ内アプリ（動く人）→ タスクロール
    - S3にファイルアップロード
    - SQSにメッセージ送信

Lambda:
  起動も実行も同じ → ロール1つだけ
```

### なぜ知っておく必要があるか
- トラブル時にどこを直せばいいか分かる
- 「コンテナが起動しない」→ タスク実行ロールを確認
- 「S3にアクセスできない」→ タスクロールを確認

---

## 実体験との紐付け
| 問題 | 原因 | レイヤー |
|------|------|---------|
| LambdaからSQS送信できない | sqs:SendMessageがない | IAM権限ポリシー |
| ECSはS3にアクセスできる | タスクロールにs3:PutObjectあり | IAM権限ポリシー |
| 外部からECSに直接アクセス不可 | ALBからのみ許可 | セキュリティグループ |

---

## 復習ポイント
- [ ] セキュリティグループとIAMの違いを説明できるか？
- [ ] ポリシーのJSON（Effect/Action/Resource）を読めるか？
- [ ] 信頼ポリシー（Principal/sts:AssumeRole）の意味を説明できるか？
- [ ] タスク実行ロールとタスクロールの違いを説明できるか？
- [ ] 「LambdaにSendMessage権限がなかった」問題を、IAMの知識で説明できるか？
