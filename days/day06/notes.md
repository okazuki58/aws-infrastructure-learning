# Day 6: ECS運用 - デプロイ・スケーリング・トラブルシューティング

- 学習日: 2026-04-16
- 所要時間: 約25分

## セクション1: CloudWatch Logsでのログ調査

### 基本コマンド3パターン
```bash
# ① 最新ログを見る（直近5分）
aws logs tail /ecs/dandori-poc-odakyu-ag-app --since 5m --format short

# ② エラーだけ絞り込む
aws logs tail /ecs/dandori-poc-odakyu-ag-app --since 30m --format short | grep -i error

# ③ 特定キーワードで絞り込む
aws logs tail /ecs/dandori-poc-odakyu-ag-app --since 30m --format short | grep "Killed"
```

### キーワード逆引き表
| 探したいもの | grep するキーワード |
|---|---|
| プロセス死亡 | `Killed` |
| コンテナ起動失敗 | `CannotPull`, `error` |
| アプリエラー | `ERROR`, `error` |
| 特定API | `temp-upload`, `confirm` |
| メモリ使用量 | `REPORT`（Lambdaの場合） |

### 実体験で発見した3つの問題
1. **「Killed」** → OOM Kill。メモリ不足でプロセス強制終了 → メモリ増量で対処
2. **「CannotPullContainerError: dummy: not found」** → イメージタグが間違い → 正しいrevisionに戻した
3. **「proxyClientMaxBodySize: Unrecognized key」** → Next.js設定が無効 → 直接の問題ではなかったが設定が効いていないことが判明

---

## セクション2: よくあるエラーパターン

### パターン1: 502 Bad Gateway
```
原因: ECSタスクが死んでいる、または起動していない
調査手順:
  1. aws ecs describe-services でrunning数を確認
  2. running=0 → タスクが起動していない
  3. events を見て CannotPullContainerError 等がないか確認
  4. aws logs tail でログを確認
```

### パターン2: 504 Gateway Timeout
```
原因: ECSが応答を返す前にALBのidle_timeoutを超えた
調査手順:
  1. ALBのidle_timeout設定を確認
  2. ECSログで処理時間を確認
  3. OOM Killが起きていないか確認（Killed）
```

### パターン3: AccessDenied
```
原因: IAMロールに必要な権限がない
調査手順:
  1. エラーメッセージから「何の操作」が拒否されたか確認
  2. 該当ロールのポリシーを確認
  3. 必要な権限を追加
```

### 実体験: AccessDeniedの例
```
LambdaからSQS SendMessageがAccessDenied
  → エラーログ: "sqs:sendmessage ... no identity-based policy allows"
  → Lambdaロールのポリシーを確認 → SendMessageがなかった
  → iam.tfに追加 → terraform apply で解決
```

---

## セクション3: トラブルシューティングの手順

### 「画面が真っ白」と報告されたら
```
Step 1: サービスの状態確認
  aws ecs describe-services で desired / running を確認
  → running=0 → タスクが動いていない（502パターン）
  → running=1 → タスクは動いているがアプリに問題

Step 2: ログ確認
  aws logs tail でエラーを探す
  → 「Killed」→ OOM Kill
  → 「Error」→ アプリのバグ
  → 何もない → フロントエンドの問題かも

Step 3: 実際にアクセスしてみる
  curl -s -o /dev/null -w "%{http_code}" https://ドメイン/
  → 502 → ECSが死んでいる
  → 500 → サーバーエラー
  → 200 → サーバーは正常、JSのエラーかも
```

### ポイント
- **大きいところから切り分ける**（いきなりコードを読まない）
- まず「サーバーが動いているか」→「アプリが応答するか」→「何のエラーか」の順

---

## セクション4: ECS運用コマンド集

### 状態確認系
```bash
# サービスの状態
aws ecs describe-services --cluster <cluster> --services <service>

# 実行中のタスク
aws ecs list-tasks --cluster <cluster> --service-name <service>

# タスク定義の詳細
aws ecs describe-task-definition --task-definition <name>:<revision>
```

### 操作系
```bash
# タスクの再起動（同じタスク定義で）
aws ecs update-service --cluster <cluster> --service <service> --force-new-deployment

# 新しいタスク定義でデプロイ
aws ecs update-service --cluster <cluster> --service <service> --task-definition <name>:<revision>

# 安定するまで待機
aws ecs wait services-stable --cluster <cluster> --services <service>
```

### ログ系
```bash
# 直近のログ
aws logs tail /ecs/<app-name> --since 5m --format short

# エラーのみ
aws logs tail /ecs/<app-name> --since 30m --format short | grep -i error
```

---

## セクション5: デプロイの流れ（CI/CD vs 手動）

### 通常デプロイ（CI/CD自動）
```
1. developにマージ
2. GitHub Actionsが起動
3. Dockerイメージをビルド → ECRにpush
4. ECSのタスク定義を新イメージで更新
5. ECSサービスをforce-new-deployment
6. ローリングデプロイで無停止入れ替え
```

### 手動対応が必要なケース
```
- SSMパラメータ変更 → ECS再起動
- タスク定義のrevision指定 → 手動ロールバック
- Lambdaの環境変数変更 → terraform apply
- IAMポリシー変更 → terraform apply
```

### 使い分け
- **コードの変更** → CI/CDで自動デプロイ
- **インフラの設定変更** → 手動（Terraform or AWSコンソール）

---

## 復習ポイント
- [ ] ECSのログを見るコマンドを3パターン書けるか？
- [ ] 502, 504, AccessDeniedの原因と調査手順を説明できるか？
- [ ] 「画面が真っ白」と言われたら何から調べるか説明できるか？
- [ ] force-new-deploymentとタスク定義指定のデプロイの違いを説明できるか？
- [ ] CI/CDで自動化されているものと手動対応が必要なものを区別できるか？
