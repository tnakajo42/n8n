# n8n + Qdrant on Google Cloud (Production Ready)

This repository provides a "One-Prompt Deployment" guide to set up a professional-grade automation environment using **n8n** and **Qdrant** on Google Cloud Platform.

## 🚀 Architecture Highlights

- **n8n (Cloud Run)**: Multi-container setup with an **nginx sidecar** to solve Origin header issues and optimize SSE (Server-Sent Events) for real-time updates.
- **Qdrant (Cloud Run)**: Vector database for AI workflows, persisted via **GCS Bucket** (Cloud Storage FUSE).
- **Cloud SQL (PostgreSQL)**: Scalable managed database for n8n metadata.
- **Security**: Database passwords managed via **Secret Manager**, SSL enforced, and deletion protection enabled.

## 🛠️ Why this setup?

- **Solved: Origin Header Issues**: Since n8n v1.87+, strict Origin validation often fails on Cloud Run. This setup uses a sidecar proxy to inject the correct headers.
- **Solved: WebSocket Instability**: Configured to use `SSE` push backend, which is significantly more stable on Cloud Run than WebSockets.
- **Solved: Cold Start Timeouts**: Uses `min-instances: 1` and `db-g1-small` to prevent common migration and connection timeouts found in cheaper tiers.

## 📦 How to Use

1. **Clone this repo** (or just copy the prompt).
2. Open `SETUP_PROMPT.md`.
3. Fill in your `GCP_PROJECT_ID`, `GCP_ACCOUNT`, and `REGION`.
4. Copy the prompt body and paste it into **Claude Code** or any gcloud-integrated AI agent.
5. Wait ~10 minutes for the automated deployment to finish.

## ⚠️ Requirements

- A Google Cloud Project with billing enabled.
- `gcloud` CLI installed and authenticated.
- **Claude Code** (recommended) or a terminal where you can run the generated commands.

## 📄 License
MIT
