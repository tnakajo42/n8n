# 使い方

1. プロンプト内の3箇所を書き換える：
   - {{GCP_PROJECT_ID}} → GCPプロジェクトID
   - {{GCP_ACCOUNT}} → gcloudログイン済みアカウント
   - {{REGION}} → リージョン（asia-northeast1 等）
2. gcloud auth login を済ませる
3. Claude Code に「## プロンプト本文」以降を貼り付けて実行

---

## 書き換え箇所（3つだけ）

| 変数 | 説明 | 例 |
|------|------|----|
| `{{GCP_PROJECT_ID}}` | GCPプロジェクトID | `my-n8n-project` |
| `{{GCP_ACCOUNT}}` | gcloud認証済みアカウント | `user@example.com` |
| `{{REGION}}` | デプロイリージョン | `asia-northeast1` |

---

## プロンプト本文 **(Just copy & paste this)**

```
【指示】
Google Cloud上に、本番運用を見据えた「n8n」と「Qdrant」の環境を構築してください。
n8n は nginx Origin Proxy サイドカーとのマルチコンテナ構成で Cloud Run にデプロイします。
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
- artifactregistry.googleapis.com
- cloudbuild.googleapis.com
- sheets.googleapis.com
- drive.googleapis.com

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

## Step 4: Secret Manager にDBパスワードを格納
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

## Step 5: Qdrant 用 GCS バケット作成とIAM設定
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

## Step 6: Qdrant を Cloud Run にデプロイ
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
- デプロイ後、発行されたQdrant URLを控える

## Step 7: n8n を Cloud Run にデプロイ（初回: シングルコンテナ）
まずシングルコンテナでn8nをデプロイし、Cloud Run URLを取得する。
このURLは後のnginxサイドカー構成で必要になる。

重要: イメージは docker.io/n8nio/n8n:latest を使用すること。
（docker.n8n.io/n8nio/n8n は Cloud Run の許可レジストリ外でエラーになる）

以下の設定でデプロイ:
- ポート: 5678
- メモリ: 2Gi
- CPU: 2
- min-instances: 1
- max-instances: 2
- --allow-unauthenticated
- --cpu-boost（起動高速化）
- --timeout=300
- --session-affinity
- Cloud SQL接続: --add-cloudsql-instances={{GCP_PROJECT_ID}}:{{REGION}}:n8n-db

環境変数:
- DB_TYPE=postgresdb
- DB_POSTGRESDB_DATABASE=n8n
- DB_POSTGRESDB_USER=n8n
- DB_POSTGRESDB_HOST=/cloudsql/{{GCP_PROJECT_ID}}:{{REGION}}:n8n-db
- DB_POSTGRESDB_CONNECTION_TIMEOUT=60000
- DB_POSTGRESDB_POOL_SIZE=5
- DB_POSTGRESDB_POOL_EXTRA={"idleTimeoutMillis":30000}
- GENERIC_TIMEZONE=Asia/Tokyo
- TZ=Asia/Tokyo
- N8N_SECURE_COOKIE=false
- N8N_ENDPOINT_HEALTH=n8nhealth
- N8N_PUSH_BACKEND=sse
- N8N_PROTOCOL=https
- N8N_PROXY_HOPS=1

シークレット参照:
- --update-secrets=DB_POSTGRESDB_PASSWORD=n8n-db-password:latest

デプロイ後、発行されたn8n URLを控える（例: https://n8n-XXXXXXXXX.{{REGION}}.run.app）。
WEBHOOK_URL, N8N_EDITOR_BASE_URL, N8N_HOST はStep 9のマルチコンテナYAMLで設定するため、
ここでは設定不要。

n8nが起動してヘルスチェックに応答するまで待つ:
  curl -s -o /dev/null -w "%{http_code}" https://<n8n-url>/n8nhealth
200が返れば起動完了。503の場合は30秒待ってリトライ（初回はマイグレーション実行で最大5分かかる）。

## Step 8: nginx Origin Proxy イメージのビルド
n8n v1.87+ は Origin ヘッダーの検証が厳格化されており、Cloud Run のロードバランサーは
Origin ヘッダーを正しく転送しない。これを解決するため、nginx リバースプロキシをサイドカーとして
配置し、正しい Origin ヘッダーを注入する。

### 8-1: Artifact Registry リポジトリ作成
gcloud artifacts repositories create n8n-nginx \
  --repository-format=docker \
  --location={{REGION}} \
  --project={{GCP_PROJECT_ID}}

### 8-2: nginx 設定ファイル作成
/tmp/n8n-nginx/nginx.conf を以下の内容で作成する。
重要: 下記の {{N8N_URL}} 部分は Step 7 で取得した実際のn8n Cloud Run URL に置換すること。

--- nginx.conf ここから ---
# Map for WebSocket upgrade (must be in http context, outside server block)
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

server {
    listen 8080;
    server_name _;

    # OAuth2 URLs can be very long (scopes, state, etc.)
    large_client_header_buffers 4 32k;
    client_max_body_size 16m;
    proxy_buffer_size 32k;
    proxy_buffers 4 32k;

    # SSE push endpoint - MUST disable buffering for real-time events
    location /rest/push {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;

        proxy_set_header Origin 'https://{{N8N_URL}}';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;

        proxy_buffering off;
        proxy_cache off;
        chunked_transfer_encoding off;
        proxy_set_header Connection '';
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;
    }

    # Default - proxy all other requests
    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;

        proxy_set_header Origin 'https://{{N8N_URL}}';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        proxy_read_timeout 300s;
        proxy_send_timeout 300s;
    }
}
--- nginx.conf ここまで ---

### 8-3: Dockerfile 作成
/tmp/n8n-nginx/Dockerfile を以下の内容で作成する:

--- Dockerfile ここから ---
FROM nginx:alpine
RUN rm /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 8080
--- Dockerfile ここまで ---

### 8-4: イメージのビルドとプッシュ
cd /tmp/n8n-nginx
gcloud builds submit \
  --tag {{REGION}}-docker.pkg.dev/{{GCP_PROJECT_ID}}/n8n-nginx/nginx-origin-proxy:latest \
  --project={{GCP_PROJECT_ID}}

## Step 9: n8n をマルチコンテナ構成で再デプロイ
nginx サイドカーを含むマルチコンテナ構成のYAMLファイルを作成し、gcloud run services replace で適用する。

重要: 以下のYAML内の3つのURLプレースホルダーを実際の値に置換すること:
- {{N8N_URL}}: Step 7 で取得したn8n Cloud Run URL（https://を含む完全なURL）
- {{N8N_HOST}}: n8n Cloud Run URLのホスト名部分（https://を除いたもの、例: n8n-XXXXXXXXX.asia-northeast1.run.app）
- {{QDRANT_URL}}: Step 6 で取得したQdrant Cloud Run URL（https://を含む完全なURL）

/tmp/n8n-multicontainer.yaml を以下の内容で作成:

--- YAML ここから ---
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: n8n
  labels:
    cloud.googleapis.com/location: {{REGION}}
  annotations:
    run.googleapis.com/launch-stage: BETA
spec:
  template:
    metadata:
      annotations:
        run.googleapis.com/container-dependencies: '{"nginx-proxy":["n8n"]}'
        run.googleapis.com/sessionAffinity: 'true'
        run.googleapis.com/startup-cpu-boost: 'true'
        run.googleapis.com/cloudsql-instances: '{{GCP_PROJECT_ID}}:{{REGION}}:n8n-db'
        autoscaling.knative.dev/minScale: '1'
        autoscaling.knative.dev/maxScale: '2'
    spec:
      containerConcurrency: 80
      containers:
        - name: nginx-proxy
          image: {{REGION}}-docker.pkg.dev/{{GCP_PROJECT_ID}}/n8n-nginx/nginx-origin-proxy:latest
          ports:
            - name: http1
              containerPort: 8080
          resources:
            limits:
              memory: 256Mi
              cpu: 500m
          startupProbe:
            httpGet:
              path: /n8nhealth
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 30

        - name: n8n
          image: docker.io/n8nio/n8n:latest
          env:
            - name: DB_TYPE
              value: postgresdb
            - name: DB_POSTGRESDB_DATABASE
              value: n8n
            - name: DB_POSTGRESDB_USER
              value: n8n
            - name: DB_POSTGRESDB_HOST
              value: /cloudsql/{{GCP_PROJECT_ID}}:{{REGION}}:n8n-db
            - name: DB_POSTGRESDB_CONNECTION_TIMEOUT
              value: '60000'
            - name: DB_POSTGRESDB_POOL_SIZE
              value: '5'
            - name: DB_POSTGRESDB_POOL_EXTRA
              value: '{"idleTimeoutMillis":30000}'
            - name: DB_POSTGRESDB_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: latest
                  name: n8n-db-password
            - name: GENERIC_TIMEZONE
              value: Asia/Tokyo
            - name: TZ
              value: Asia/Tokyo
            - name: N8N_SECURE_COOKIE
              value: 'false'
            - name: WEBHOOK_URL
              value: '{{N8N_URL}}'
            - name: N8N_EDITOR_BASE_URL
              value: '{{N8N_URL}}'
            - name: N8N_HOST
              value: '{{N8N_HOST}}'
            - name: N8N_PROTOCOL
              value: https
            - name: N8N_PROXY_HOPS
              value: '1'
            - name: N8N_ENDPOINT_HEALTH
              value: n8nhealth
            - name: N8N_PUSH_BACKEND
              value: sse
            - name: QDRANT_URL
              value: '{{QDRANT_URL}}'
          resources:
            limits:
              memory: 2Gi
              cpu: '2'
          startupProbe:
            httpGet:
              path: /n8nhealth
              port: 5678
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 30
--- YAML ここまで ---

YAMLファイル作成後、以下で適用:
  gcloud run services replace /tmp/n8n-multicontainer.yaml \
    --region={{REGION}} \
    --project={{GCP_PROJECT_ID}}

適用後、n8nサービスが未認証アクセスを許可しているか確認し、必要に応じて再設定:
  gcloud run services set-iam-policy n8n /dev/stdin --region={{REGION}} --project={{GCP_PROJECT_ID}} <<'POLICY'
  bindings:
  - members:
    - allUsers
    role: roles/run.invoker
  POLICY

## Step 10: 動作確認
以下を確認し、結果を報告:

1. n8n ヘルスチェック:
   curl -s -o /dev/null -w "%{http_code}" https://<n8n-url>/n8nhealth
   → 200が返ること（/healthz ではない。N8N_ENDPOINT_HEALTH=n8nhealth でパスを変更済み）
   初回はマイグレーション実行のため最大5分かかる。503が返る場合は30秒待ってリトライ。

2. SSE Push エンドポイント確認:
   curl -s -o /dev/null -w "%{http_code}" https://<n8n-url>/rest/push?pushRef=test
   → 401が返ること（認証が必要なため401が正常。500が返る場合はnginxのOrigin注入が機能していない）

3. Qdrant ヘルスチェック:
   curl -s https://<qdrant-url>/
   → バージョン情報のJSONが返ること

4. Cloud SQLのauthorizedNetworksが空であること（外部直接アクセス不可）

5. Secret Managerのシークレットが正しく参照されていること（n8nのログにDB接続成功が出ること）

## Step 11: 最終出力
構築完了後、以下を出力:
- n8nのアクセスURL
- QdrantのURL
- 「ブラウザでn8nのURLを開き、管理者アカウントを作成してください。第三者にアカウントを先に作られないよう、速やかに実施してください。」というメッセージ

【注意事項まとめ】
- db-f1-micro は絶対に使わない（n8nのTypeORMが頻繁にタイムアウトする）
- docker.n8n.io はCloud Runで使えない。docker.io/n8nio/n8n を使う
- n8n のヘルスチェックパスは N8N_ENDPOINT_HEALTH で /n8nhealth に変更（GCPが /healthz をLBレベルで予約しているため）
- 両方のコンテナ（nginx-proxy, n8n）の startupProbe は /n8nhealth を使うこと
- nginx サイドカーは n8n v1.87+ の Origin ヘッダー検証問題を解決するために必須
- nginx の large_client_header_buffers は OAuth2 の長いURL（スコープ、state等）に対応するため 32k に設定
- N8N_PUSH_BACKEND=sse は Cloud Run での SSE リアルタイム通信に必須（WebSocket は Cloud Run 上で不安定）
- N8N_PROXY_HOPS=1 で Cloud Run のプロキシヘッダーを信頼する
- startupProbe のfailureThreshold を大きくしないと、マイグレーション中にコンテナが再起動ループする
- min-instances=0 だとコールドスタート時にDB接続タイムアウトが起きやすい。n8nは min-instances=1 推奨
- WEBHOOK_URL と N8N_EDITOR_BASE_URL は必ず設定する（未設定だとWebhookが正しく動かない）
- N8N_SECURE_COOKIE=false は必須（Cloud Runのプロキシ経由でSecure Cookie判定が狂うため）
- DBパスワードは必ずSecret Manager経由で参照する（環境変数に平文で設定しない）
- Cloud SQLの削除保護を有効にしておくこと（誤操作によるデータ消失を防止）
- Qdrantのデータ永続化にはGCSボリュームマウントを使う（Cloud Runのローカルストレージはエフェメラル）
- デプロイ後の再ログインが必要（コンテナ再起動でインメモリセッションがクリアされるため）
```

---

## セットアップ後: OAuth2 設定（Google Sheets等で必要）

以下はプロンプト外の手動手順です。n8n から Google Sheets / Google Drive 等を使う場合に必要です。

### 手順

1. [Google Cloud Console](https://console.cloud.google.com/) にアクセス
2. **APIs & Services** → **Credentials** に移動
3. **+ CREATE CREDENTIALS** → **OAuth client ID** を選択
4. アプリケーションの種類: **Web application**
5. 名前: 任意（例: `n8n OAuth2`）
6. **Authorized redirect URIs** に以下を追加:
   ```
   https://<n8n-url>/rest/oauth2-credential/callback
   ```
   （`<n8n-url>` は実際のn8n Cloud Run URLに置換）
7. **CREATE** をクリック
8. 表示された **Client ID** と **Client Secret** をコピー

### n8n側の設定

1. n8n のエディタを開く
2. **Credentials** → **Add Credential** → **Google Sheets OAuth2 API**（または該当するサービス）
3. 上記でコピーした **Client ID** と **Client Secret** を入力
4. **Sign in with Google** をクリックしてOAuth認証フローを完了

> **注意**: OAuth consent screen が未設定の場合、先に **APIs & Services** → **OAuth consent screen** でセットアップが必要です。テスト段階では「External」→テストユーザーに自分のアカウントを追加してください。

---

## 使わなくなった時の停止コマンド

```bash
# Cloud SQL 停止
gcloud sql instances patch n8n-db --activation-policy=NEVER --project={{GCP_PROJECT_ID}}

# Cloud Run を0にスケール（n8n）
gcloud run services update n8n --min-instances=0 --region={{REGION}} --project={{GCP_PROJECT_ID}}

# Cloud Run を0にスケール（Qdrant）
gcloud run services update qdrant --min-instances=0 --region={{REGION}} --project={{GCP_PROJECT_ID}}
```

### 再開

```bash
gcloud sql instances patch n8n-db --activation-policy=ALWAYS --project={{GCP_PROJECT_ID}}
gcloud run services update n8n --min-instances=1 --region={{REGION}} --project={{GCP_PROJECT_ID}}
gcloud run services update qdrant --min-instances=1 --region={{REGION}} --project={{GCP_PROJECT_ID}}
```

---

## 完全削除コマンド（注意: データがすべて消えます）

```bash
# 削除保護を解除してからCloud SQLを削除
gcloud sql instances patch n8n-db --no-deletion-protection --project={{GCP_PROJECT_ID}}
gcloud sql instances delete n8n-db --project={{GCP_PROJECT_ID}}

# Cloud Run サービス削除
gcloud run services delete n8n --region={{REGION}} --project={{GCP_PROJECT_ID}}
gcloud run services delete qdrant --region={{REGION}} --project={{GCP_PROJECT_ID}}

# Secret Manager シークレット削除
gcloud secrets delete n8n-db-password --project={{GCP_PROJECT_ID}}

# GCS バケット削除
gcloud storage rm -r gs://{{GCP_PROJECT_ID}}-qdrant-data

# Artifact Registry リポジトリ削除（nginx Origin Proxy イメージ）
gcloud artifacts repositories delete n8n-nginx --location={{REGION}} --project={{GCP_PROJECT_ID}}
```
