# Instructions
1. Replace the 3 placeholders: `{{GCP_PROJECT_ID}}`, `{{GCP_ACCOUNT}}`, and `{{REGION}}`.
2. Ensure you have run `gcloud auth login`.
3. Paste the text below into your AI agent.

---

## PROMPT BODY

```
【Task】
Deploy a production-ready "n8n" and "Qdrant" environment on Google Cloud. 
n8n must be deployed to Cloud Run using a multi-container setup with an nginx Origin Proxy sidecar.
Execute all steps using gcloud commands sequentially. Proceed automatically without asking for confirmation.

【Context】
- GCP Account: {{GCP_ACCOUNT}}
- Project ID: {{GCP_PROJECT_ID}}
- Region: {{REGION}}
- gcloud auth login: Completed

【Steps】

## Step 1: Initial Config
gcloud config set account {{GCP_ACCOUNT}}
gcloud config set project {{GCP_PROJECT_ID}}

## Step 2: Enable APIs
Enable: run.googleapis.com, sqladmin.googleapis.com, sql-component.googleapis.com, compute.googleapis.com, secretmanager.googleapis.com, artifactregistry.googleapis.com, cloudbuild.googleapis.com, sheets.googleapis.com, drive.googleapis.com.

## Step 3: Cloud SQL (PostgreSQL)
Create instance "n8n-db":
- Version: POSTGRES_15
- Tier: db-g1-small (Do NOT use db-f1-micro due to performance issues)
- Storage: 10GB HDD, Deletion Protection: Enabled
- SSL: Required (gcloud sql instances patch --require-ssl)
- Flags: tcp_keepalives_idle=60, tcp_keepalives_interval=30, tcp_keepalives_count=3, idle_in_transaction_session_timeout=300000, log_lock_waits=on, log_min_duration_statement=3000
- Backups: 03:00 UTC, PITR: Enabled, Retention: 7 days.

After creation:
- Create database "n8n".
- Create user "n8n" with a password generated via `openssl rand -base64 18`.

## Step 4: Secret Manager
Store the generated password in Secret Manager as "n8n-db-password".
Grant `roles/secretmanager.secretAccessor` to the default Compute Engine service account.

## Step 5: Qdrant GCS Setup
Create bucket `gs://{{GCP_PROJECT_ID}}-qdrant-data` for Qdrant persistence.
Grant `roles/storage.objectAdmin` to the service account.

## Step 6: Deploy Qdrant
- Image: qdrant/qdrant:latest
- Resources: 1 CPU, 1Gi RAM, min-instances: 1
- Env: gen2 execution environment
- Volume: Mount the GCS bucket to `/qdrant/storage`.

## Step 7: Initial n8n Deploy (Single Container)
Deploy `docker.io/n8nio/n8n:latest` (Do NOT use docker.n8n.io).
- Port: 5678, Resources: 2 CPU, 2Gi RAM, min-instances: 1
- Features: cpu-boost, session-affinity, cloud-sql-instance link.
- Secret: Link DB_POSTGRESDB_PASSWORD to n8n-db-password:latest.
- Capture the generated Cloud Run URL.

## Step 8: nginx Origin Proxy Build
n8n v1.87+ requires strict Origin header matching. We will use nginx to inject this.
1. Create Artifact Registry "n8n-nginx".
2. Create `/tmp/n8n-nginx/nginx.conf` with:
   - Port: 8080
   - Proxy to `localhost:5678`
   - Header: `proxy_set_header Origin 'https://<CAPTURED_N8N_URL>'`
   - Buffers: `large_client_header_buffers 4 32k` (for large OAuth URLs)
   - SSE Optimization: Disable buffering for `/rest/push`.
3. Create Dockerfile, build via Cloud Build, and push to Registry.

## Step 9: Multi-Container Redeploy
Create and apply a Knative YAML (`gcloud run services replace`):
- Container 1: `nginx-proxy` (Port 8080)
- Container 2: `n8n` (Port 5678)
- Env Vars: `N8N_PUSH_BACKEND=sse`, `N8N_PROTOCOL=https`, `N8N_PROXY_HOPS=1`, `WEBHOOK_URL`, `N8N_EDITOR_BASE_URL`.
- Startup Probes: Both containers should probe `/n8nhealth`.

## Step 10: Validation & Summary
1. Check n8n health (/n8nhealth == 200).
2. Check SSE (/rest/push?pushRef=test == 401).
3. Check Qdrant root URL.
4. Output the final URLs and a security warning to create the admin account immediately.
```
