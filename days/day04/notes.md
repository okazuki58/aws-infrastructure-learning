# Day 4: ECS/Fargate基礎 - タスク定義・サービス・クラスター

- 学習日: 2026-04-07
- 所要時間: 約30分

## セクション1: ECSの3つの概念

### 料理の例え
```
クラスター = キッチン（作業場所）
タスク定義 = レシピ（設計図）
サービス   = シェフ（管理者。レシピ通りにタスクを作り、常に動いている状態を維持）
タスク     = 料理（実際に動いているコンテナ）
```

### DandoriAI POC環境の構成
```
dandori-poc-cluster（キッチン）
├── nhk-ep: app-service + weaviate
├── odakyu-ag: app-service + weaviate
└── demo: app-service + weaviate
```
- 1つのクラスターに6つのサービス
- 各テナントがアプリ用とWeaviate用の2つのサービスを持つ

### 確認コマンド
```bash
aws ecs list-services --cluster dandori-poc-cluster --query "serviceArns[*]" --output table
```

---

## セクション2: タスク定義（設計図）

### タスク定義に含まれるもの
```
1. コンテナの基本設定
   - イメージ（どのコンテナを動かすか）
   - CPU / メモリ（どのくらいのスペックか）
   - ポート（何番ポートで待ち受けるか）

2. 環境設定
   - environment（直接設定する環境変数）
   - secrets（SSMから取得する環境変数）

3. ログ設定（CloudWatch Logs出力先）
4. ネットワーク設定（Fargate/EC2、CPUアーキテクチャ）
5. IAMロール（タスク実行ロール、タスクロール）
```

### revision（バージョン管理）
- タスク定義は変更するたびにrevision番号が上がる
- 古いrevisionに戻すことも可能

### 実体験: revision 6 → 7 → 8
```
revision 6: CPU 512, メモリ 1024MB, イメージ main-d3db09e
  → メモリ不足でOOM Kill

revision 7: CPU 1024, メモリ 2048MB, イメージ dummy
  → イメージタグが間違っていてコンテナpull失敗

revision 8: CPU 1024, メモリ 2048MB, イメージ main-d3db09e
  → 正常動作
```

### 確認コマンド
```bash
# タスク定義の詳細確認
aws ecs describe-task-definition --task-definition dandori-poc-odakyu-ag-app-task:8 \
  --query "taskDefinition.{cpu:cpu,memory:memory,image:containerDefinitions[0].image}" \
  --output json
```

---

## セクション3: environment vs secrets

### 違い
```
environment（直接設定）:
  タスク定義に値がそのまま書かれている
  → 誰でも値が見える
  例: MIDDLEWARE_ORIGIN=http://127.0.0.1:3000

secrets（SSM経由）:
  SSM Parameter StoreのARNだけが書かれている
  → 起動時にECSがSSMから値を取得して環境変数にセット
  → タスク定義を見てもAPIキーの値は見えない
  例: DATABASE_URL → arn:aws:ssm:.../DATABASE_URL
```

### ほとんどがsecrets（36個）でenvironment（2個）が少ない理由
1. **セキュリティ**: APIキーやDB接続情報をタスク定義に直書きしたくない
2. **柔軟性**: SSMの値を変えてECS再起動するだけで変更可能。タスク定義を作り直す必要なし

### 注意点
- SSMの値を変更してもECSには即座に反映されない
- ECSは起動時に1回だけSSMから読み込むため、再起動（force-new-deployment）が必要

---

## セクション4: Fargateとは

### ECSの2つの動かし方
```
EC2モード:
  自分でサーバー（EC2）を用意してその上にコンテナを載せる
  → サーバーのOS更新、スケーリングは自分で管理
  → 安いが手間がかかる

Fargateモード:
  サーバーの管理はAWSにおまかせ
  → タスク定義でCPU/メモリを指定するだけ
  → 高いが手間がかからない
```

- DandoriAIはFargateを使用
- スペック変更 = タスク定義のCPU/メモリの数字を変えるだけ

---

## セクション5: サービスのデプロイフロー

### force-new-deploymentの流れ
```
1. 新しいタスクBを起動           → running: 1, pending: 1
2. ヘルスチェック（/api/health）通過を待つ
3. ALBがトラフィックをタスクBに切り替え
4. タスクAへの接続をdraining（既存接続の完了を待つ）
5. タスクAを停止               → running: 1（タスクBのみ）
```

### ECSイベントの読み方
```
"has started 1 tasks"              → 新タスク起動
"registered 1 targets"             → ALBに登録
"has begun draining connections"   → 旧タスクの接続排出開始
"deregistered 1 targets"           → ALBから旧タスク解除
"has stopped 1 running tasks"      → 旧タスク停止
"has reached a steady state"       → 安定状態（完了）
```

### 実体験: revision 7のデプロイ失敗
- イメージタグが`dummy` → pull失敗 → ヘルスチェックに永遠に通らない
- `CannotPullContainerError`がイベントに出続ける
- revision 6に戻してforce-new-deploymentで復旧

### 確認コマンド
```bash
# サービスの状態確認
aws ecs describe-services --cluster dandori-poc-cluster \
  --services dandori-poc-odakyu-ag-app-service \
  --query "services[0].{desired:desiredCount,running:runningCount,taskDef:taskDefinition,events:events[0].message}" \
  --output json
```

---

## 復習ポイント
- [ ] クラスター・タスク定義・サービス・タスクの関係を説明できるか？
- [ ] タスク定義に含まれる設定項目を5つ以上挙げられるか？
- [ ] environment と secrets の違いを説明できるか？
- [ ] SSMの値を変えた後に何が必要か説明できるか？
- [ ] FargateとEC2モードの違いを説明できるか？
- [ ] force-new-deploymentの流れを説明できるか？
- [ ] ECSイベントのメッセージを読んで何が起きているか判断できるか？
