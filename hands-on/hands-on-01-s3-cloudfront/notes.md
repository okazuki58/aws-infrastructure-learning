# Hands-on Day 1: S3 + CloudFront で静的サイト配信

- 実施日: 2026-04-22 〜 2026-04-23
- 所要時間: 約2時間（アカウント作成含む）
- SAA頻出度: ★★★（超頻出）

## 目的

S3 + CloudFront + OAC を使って、静的サイトを高速・安全に配信する
SAAでの設計判断問題に答えられるレベルで仕組みを理解する

## 事前準備

### AWSアカウント作成
- ルートアカウント作成
- MFA設定（必須）
- AWS Budgets で月額5,000円のアラート設定

### IAMユーザー作成
- ユーザー名: okazuki
- 権限: AdministratorAccess
- MFA設定
- アクセスキー作成 → CSVでダウンロード

### AWS CLI設定
```bash
aws configure --profile learning
# Access Key ID, Secret Access Key, region=ap-northeast-1, format=json

aws sts get-caller-identity --profile learning
```

---

## 作ったもの

```
ユーザー
  ↓ https://dxxxxx.cloudfront.net
CloudFront（エッジロケーションにキャッシュ配信）
  ↓ OAC経由
S3バケット（非公開）
  └─ index.html
```

## セットアップ手順

### Step 1: S3バケット作成
- バケット名: `okazuki-learning-cloudfront-demo-20260422`
- リージョン: ap-northeast-1
- パブリックアクセス: すべてブロック（重要）
- 暗号化: SSE-S3

### Step 2: HTMLファイルをアップロード
- `index.html` をS3にアップロード
- 「Access Denied」になることを確認（非公開が正しい状態）

### Step 3: CloudFrontディストリビューション作成
- Origin: 作成したS3バケット
- Origin access: OAC（Origin Access Control）を作成
- WAF: Do not enable security protections
- Default root object: `index.html`（重要、忘れがち）

### Step 4: S3バケットポリシーの設定
CloudFrontが生成したポリシーをコピーしてS3のバケットポリシーに貼り付け
```json
{
  "Version": "2008-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "cloudfront.amazonaws.com" },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::okazuki-learning-cloudfront-demo-20260422/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::XXXX:distribution/XXXXX"
      }
    }
  }]
}
```

### Step 5: 動作確認
- `https://xxx.cloudfront.net/` → 403（Default root object未設定だった）
- `https://xxx.cloudfront.net/index.html` → 200（キャッシュ配信成功）
- Default root object を `index.html` に設定 → `/` でもアクセス可能に

---

## つまづきポイント

### 1. Default root object の設定忘れ
CloudFront作成時のウィザードに明示的な項目がなく、後から設定する必要があった
`/` でアクセスされたとき何を返すかの指定

### 2. UIが更新されている
「ポリシーを直接アタッチ」のUIが新しいウィザード形式になっていた
→ 画面を見ながら対応箇所を探す必要があった

### 3. OACの仕組み
`AWS:SourceArn` で特定のCloudFrontディストリビューションに限定
→ 別のCloudFrontから勝手にアクセスされないようにする

---

## 学んだこと（SAA対策）

### CloudFrontの役割
- CDN（Content Delivery Network）
- 世界中のエッジロケーション（400+箇所）にキャッシュ
- ユーザー近くから配信 → 低レイテンシ
- HTTPS対応、DDoS対策も自動

### S3の役割（CloudFrontと組み合わせる場合）
- 非公開バケットとして使う（パブリックアクセス全ブロック）
- CloudFrontからのみアクセス可能（OACで制限）
- オリジン（本店）としての役割

### OAC（Origin Access Control）
- CloudFrontからS3へのアクセスを認証
- 旧来のOAI（Origin Access Identity）の後継、新しい推奨方式
- バケットポリシーで `AWS:SourceArn` 条件で制限

### キャッシュの仕組み
```
1回目のアクセス:
  ユーザー → CloudFront → キャッシュなし → S3からファイル取得
  → キャッシュに保存 → ユーザーに返す

2回目以降（別のユーザーでも可）:
  ユーザー → CloudFront → キャッシュあり → 即返す（S3に行かない）
```

### 料金
- S3: ストレージ + データ転送で数十円程度
- CloudFront: データ転送で数百円程度
- 個人ブログ規模なら月数百円〜1,000円程度

### Price Class
- `PriceClass_All`: 全世界（最高価格帯）
- `PriceClass_200`: 北米、ヨーロッパ、アジア、オセアニア
- `PriceClass_100`: 北米、ヨーロッパ（最安）
- 学習用なら `PriceClass_100` でOK

---

## SAA頻出パターン

### パターン1: グローバル配信
```
問題: グローバルにユーザーがいるWebアプリの静的コンテンツを高速配信したい
回答: S3 + CloudFront
理由: CDNで各地域のエッジからキャッシュ配信
```

### パターン2: セキュアな静的サイト
```
問題: S3の静的サイトを安全に配信したい
回答: S3（非公開）+ CloudFront + OAC
理由: S3を非公開にしてCloudFront経由でのみアクセス可能にする
```

### パターン3: 動的 vs 静的の判断
```
問題: コメント機能付きブログ
→ 静的＋動的の組み合わせ
→ CloudFront + S3（静的）＋ Lambda + DynamoDB（コメント）

問題: ユーザーごとにログインが必要なアプリ
→ ECS + ALB + RDS（S3 + CloudFrontだけでは不可）
```

---

## 畑の例えで振り返り

```
S3 = 畑の外にある本店の倉庫（東京にある）
CloudFront = 世界中に作られた支店ネットワーク
           （ユーザーの近くから配送してくれる）

支店が本店のコピーを持っていて、なければ本店に取りに行く
→ そのコピーを支店に置いて、次回から速く配信
```

---

## 後片付け（未実施）

S3とCloudFrontは放置してもほぼ課金されない（月数十円程度）ので、このまま残す。

削除する場合:
```bash
# CloudFrontディストリビューションを無効化→削除（反映に時間がかかる）
# S3バケットを空にしてから削除
aws s3 rm s3://okazuki-learning-cloudfront-demo-20260422 --recursive --profile learning
aws s3 rb s3://okazuki-learning-cloudfront-demo-20260422 --profile learning
```

---

## 復習ポイント
- [ ] CloudFrontが何をするサービスか説明できるか？
- [ ] S3 + CloudFront で静的配信する典型パターンを説明できるか？
- [ ] OACの役割を説明できるか？
- [ ] Default root objectの意味を説明できるか？
- [ ] 静的サイトと動的アプリの構成の違いを説明できるか？
- [ ] Price Classの違いを説明できるか？

---

## 次のハンズオン予定

Day 2: DynamoDB 基礎（RDSとの違い、NoSQLの体験）
