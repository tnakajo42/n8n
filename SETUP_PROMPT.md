# Cloud Run Production-Ready n8n & Qdrant Setup Prompt

## Instructions
1. Replace the following 3 variables in the prompt below:
   - `{{GCP_PROJECT_ID}}` → Your Google Cloud Project ID
   - `{{GCP_ACCOUNT}}` → Your gcloud authenticated account (email)
   - `{{REGION}}` → Deployment region (e.g., `us-central1`, `asia-northeast1`)
2. Ensure you have run `gcloud auth login`.
3. Copy and paste the text from "## PROMPT BODY" into **Claude Code** or your terminal-capable AI agent.

---

## PROMPT BODY

```
【Objective】
Deploy a production-ready environment for "n8n" and "Qdrant" on Google Cloud.
n8n will be deployed to Cloud Run using a multi-container setup with an nginx Origin Proxy sidecar.
Execute all steps using gcloud commands sequentially without asking for confirmation.

【Context】

GCP Account: {{GCP_ACCOUNT}}

Project ID: {{GCP_PROJECT_ID}}

Region: {{REGION}}

【Steps】

Step 1: Initialization
gcloud config set account {{GCP_ACCOUNT}}
gcloud config set project {{GCP_PROJECT_ID}}

Step 2: Enable APIs
Enable the following: run, sqladmin, compute, secretmanager, artifactregistry, cloudbuild, sheets, drive.

Step 3: Cloud SQL (PostgreSQL) Setup
Instance: n8n-db (PostgreSQL 15)

Tier: db-g1-small (Required for stability)

Storage: 10GB HDD with deletion protection enabled.

Backups: Automated (03:00 UTC), PITR enabled, 7-day retention.

SSL: Required (requireSsl=true).

Create database "n8n" and user "n8n" (generate password automatically).

Step 4: Secret Manager
Store the generated DB password as a secret named n8n-db-password.
Grant roles/secretmanager.secretAccessor to the Compute Engine default service account.

Step 5: GCS for Qdrant Persistence
Create bucket gs://{{GCP_PROJECT_ID}}-qdrant-data for Qdrant storage.
Grant roles/storage.objectAdmin to the service account.

Step 6: Deploy Qdrant to Cloud Run
Image: qdrant/qdrant:latest

Resources: 1 CPU, 1Gi RAM

Scaling: min-instances 1 (prevents data reload delay), max-instances 1

Execution Environment: gen2 (required for GCS mount)

Mount: GCS bucket to /qdrant/storage

Step 7: Initial n8n Deployment (Single Container)
Deploy docker.io/n8nio/n8n:latest to obtain the Service URL.

Resources: 2 CPU, 2Gi RAM

Config: CPU boost enabled, Session affinity enabled, Cloud SQL connection linked.

Auth: --allow-unauthenticated

Step 8: Build nginx Origin Proxy
Create an nginx sidecar to handle the strict Origin header validation in n8n v1.87+.

Create Artifact Registry repository n8n-nginx.

Generate nginx.conf injecting the {{N8N_URL}} into the Origin header.

Build and push the image via Cloud Build.

Step 9: Multi-Container Deployment
Apply a multi-container YAML configuration including:

Container 1: nginx-proxy (Port 8080)

Container 2: n8n (Port 5678)

Environment Variables: DB config, N8N_PUSH_BACKEND=sse, N8N_PROTOCOL=https, etc.

Dependencies: nginx-proxy depends on n8n.

Step 10: Validation
Verify n8n health check (/n8nhealth returns 200).

Verify SSE Push endpoint (returns 401, not 500).

Verify Qdrant response.

Step 11: Final Output
Print the n8n Access URL, Qdrant URL, and a warning to set up the admin account immediately.
```
