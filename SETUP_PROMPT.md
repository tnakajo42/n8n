```
【指示】
Google Cloud上に、本番運用を見据えた「n8n」と「Qdrant」の環境を構築してください。
以下の手順を上から順に、すべてgcloudコマンドで実行してください。
途中で確認は不要です。すべて自動で進めてください。

【前提】
- GCPアカウント: {{GCP_ACCOUNT}}
- プロジェクトID: {{GCP_PROJECT_ID}}
- リージョン: {{REGION}}
- gcloud auth login は完了済み

【手順】

## Step 1: 初期設定
gcloud config set account {{GCP_ACCOUNT}}
gcloud config set project {{GCP_PROJECT_ID}}

## Step 2: API有効化
以下のAPIを有効化:
- run.googleapis.com
- sqladmin.googleapis.com
- sql-component.googleapis.com
- compute.googleapis.com
- secretmanager.googleapis.com

## Step 3: Cloud SQL (PostgreSQL) 作成
以下の設定で作成（完了まで数分かかる）:
- インスタンス名: n8n-db
- バージョン: POSTGRES_15
- マシンタイプ: db-g1-small（重要: db-f1-microはn8nには性能不足でタイムアウトが頻発するため使わないこと）
- ストレージ: HDD 10GB
- パブリックIP有効（--assign-ip）
- edition: enterprise
- 削除保護: 有効（--deletion-protection）
- SSL必須: requireSsl=true（--database-flags ではなく gcloud sql instances patch --require-ssl で設定）

データベースフラグ（接続安定性・監視・運用向け）:
- tcp_keepalives_idle=60
- tcp_keepalives_interval=30
- tcp_keepalives_count=3
- idle_in_transaction_session_timeout=300000
- log_lock_waits=on
- log_min_duration_statement=3000

自動バックアップ設定:
- 毎日 03:00 UTC に自動バックアップ
- ポイントインタイムリカバリ（PITR）有効
- 保持期間: 7日間
- コマンド例:
  gcloud sql instances patch n8n-db \
    --backup-start-time=03:00 \
    --enable-point-in-time-recovery \
    --retained-backups-count=7 \
    --project={{GCP_PROJECT_ID}}

作成後、以下を実行:
- データベース「n8n」を作成
- ユーザー「n8n」を作成（パスワードはopenssl rand -base64 18で自動生成）

## Step 3.5: Secret Manager にDBパスワードを格納
生成したパスワードをSecret Managerに保存する（Cloud Runの環境変数に平文で持たせない）:
- シークレット名: n8n-db-password
- コマンド:
  echo -n "（生成したパスワード）" | gcloud secrets create n8n-db-password \
    --data-file=- \
    --replication-policy=automatic \
    --project={{GCP_PROJECT_ID}}

Cloud Runサービスアカウントにシークレットへのアクセス権を付与:
  gcloud secrets add-iam-policy-binding n8n-db-password \
    --member="serviceAccount:$(gcloud iam service-accounts list --filter='displayName:Compute Engine default' --format='value(email)' --project={{GCP_PROJECT_ID}})" \
    --role="roles/secretmanager.secretAccessor" \
    --project={{GCP_PROJECT_ID}}

## Step 4: Qdrant 用 GCS バケット作成とIAM設定
Qdrantのデータを永続化するためのGCSバケットを作成する:
- バケット名: {{GCP_PROJECT_ID}}-qdrant-data
- コマンド:
  gcloud storage buckets create gs://{{GCP_PROJECT_ID}}-qdrant-data \
    --location={{REGION}} \
    --project={{GCP_PROJECT_ID}}

Cloud Runサービスアカウントにバケットへのアクセス権を付与:
  gcloud storage buckets add-iam-policy-binding gs://{{GCP_PROJECT_ID}}-qdrant-data \
    --member="serviceAccount:$(gcloud iam service-accounts list --filter='displayName:Compute Engine default' --format='value(email)' --project={{GCP_PROJECT_ID}})" \
    --role="roles/storage.objectAdmin"

## Step 5: Qdrant を Cloud Run にデプロイ
- イメージ: qdrant/qdrant:latest
- ポート: 6333
- メモリ: 1Gi
- CPU: 1
- min-instances: 1（ベクトルDBはコールドスタートでデータ再読込が遅いため常時起動）
- max-instances: 1
- --allow-unauthenticated
- 実行環境: gen2（--execution-environment=gen2、GCSボリュームマウントに必要）
- GCSボリュームマウント:
  --add-volume=name=qdrant-data,type=cloud-storage,bucket={{GCP_PROJECT_ID}}-qdrant-data
  --add-volume-mount=volume=qdrant-data,mount-path=/qdrant/storage
- デプロイ後、発行されたURLを控える

## Step 6: n8n を Cloud Run にデプロイ
重要: イメージは docker.io/n8nio/n8n:latest を使用すること。
（docker.n8n.io/n8nio/n8n は Cloud Run の許可レジストリ外でエラーになる）

以下の設定でデプロイ:
- ポート: 5678
- メモリ: 2Gi
- CPU: 2
- min-instances: 1（コールドスタートでDB接続タイムアウトを防ぐため）
- max-instances: 2
- --allow-unauthenticated
- --cpu-boost（起動高速化）
- --timeout=300
- --session-affinity（WebSocketやエディタセッションの安定性向上）
- Cloud SQL接続: --add-cloudsql-instances={{GCP_PROJECT_ID}}:{{REGION}}:n8n-db

環境変数:
- DB_TYPE=postgresdb
- DB_POSTGRESDB_DATABASE=n8n
- DB_POSTGRESDB_USER=n8n
- DB_POSTGRESDB_HOST=/cloudsql/{{GCP_PROJECT_ID}}:{{REGION}}:n8n-db
- DB_POSTGRESDB_CONNECTION_TIMEOUT=60000
- GENERIC_TIMEZONE=Asia/Tokyo
- TZ=Asia/Tokyo
- N8N_SECURE_COOKIE=false
- WEBHOOK_URL=（デプロイ後に確定するn8nのCloud Run URL）
- N8N_EDITOR_BASE_URL=（同上）
- QDRANT_URL=（Step5で控えたQdrantのURL）

シークレット参照（パスワードはSecret Managerから取得）:
- --update-secrets=DB_POSTGRESDB_PASSWORD=n8n-db-password:latest

重要: デプロイ後、gcloud run services replace でKnativeのYAMLを適用し、
startupProbe を以下の設定にすること（デフォルトだとDB初期化中にコンテナが落とされる）:
- httpGet.path: /healthz
- httpGet.port: 5678
- initialDelaySeconds: 30
- periodSeconds: 10
- timeoutSeconds: 5
- failureThreshold: 30

## Step 7: 動作確認
以下を確認し、結果を報告:
1. n8nのURL（https://～.run.app）にcurlでアクセスし、HTTP 200が返ること
   - 初回はマイグレーション実行のため最大5分かかる。503が返る場合は30秒待ってリトライ。
   - ログに「Editor is now accessible via:」が出れば起動完了。
2. QdrantのURLにアクセスし、バージョン情報のJSONが返ること
3. Cloud SQLのauthorizedNetworksが空であること（外部直接アクセス不可）
4. Secret Managerのシークレットが正しく参照されていること（n8nのログにDB接続成功が出ること）

## Step 8: 最終出力
構築完了後、以下を出力:
- n8nのアクセスURL
- QdrantのURL
- 「ブラウザでn8nのURLを開き、管理者アカウントを作成してください。第三者にアカウントを先に作られないよう、速やかに実施してください。」というメッセージ

【注意事項まとめ】
- db-f1-micro は絶対に使わない（n8nのTypeORMが頻繁にタイムアウトする）
- docker.n8n.io はCloud Runで使えない。docker.io/n8nio/n8n を使う
- startupProbe のfailureThreshold を大きくしないと、マイグレーション中にコンテナが再起動ループする
- min-instances=0 だとコールドスタート時にDB接続タイムアウトが起きやすい。n8nは min-instances=1 推奨
- WEBHOOK_URL と N8N_EDITOR_BASE_URL は必ず設定する（未設定だとWebhookが正しく動かない）
- N8N_SECURE_COOKIE=false は必須（Cloud Runのプロキシ経由でSecure Cookie判定が狂うため）
- DBパスワードは必ずSecret Manager経由で参照する（環境変数に平文で設定しない）
- Cloud SQLの削除保護を有効にしておくこと（誤操作によるデータ消失を防止）
- Qdrantのデータ永続化にはGCSボリュームマウントを使う（Cloud Runのローカルストレージはエフェメラル）
```

---

## 使わなくなった時の停止コマンド

```bash
# Cloud SQL 停止
gcloud sql instances patch n8n-db --activation-policy=NEVER --project={{GCP_PROJECT_ID}}

# Cloud Run を0にスケール（n8n）
gcloud run services update n8n --min-instances=0 --region={{REGION}} --project={{GCP_PROJECT_ID}}

# Cloud Run を0にスケール（Qdrant）
gcloud run services update qdrant --min-instances=0 --region={{REGION}} --project={{GCP_PROJECT_ID}}

# 再開
gcloud sql instances patch n8n-db --activation-policy=ALWAYS --project={{GCP_PROJECT_ID}}
gcloud run services update n8n --min-instances=1 --region={{REGION}} --project={{GCP_PROJECT_ID}}
gcloud run services update qdrant --min-instances=1 --region={{REGION}} --project={{GCP_PROJECT_ID}}
```
