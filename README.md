# 🚀 One-Prompt n8n & Qdrant Production Setup on Google Cloud

Deploy a high-availability, production-ready n8n and Qdrant environment on Cloud Run with a single copy-paste prompt.

## ✨ Key Features
- **n8n v1.87+ Compatible**: Includes an nginx sidecar to handle the "Strict Origin" issues on Cloud Run.
- **Persistent Vector DB**: Qdrant with GCS (Google Cloud Storage) FUSE mount for data persistence.
- **Enterprise DB**: Cloud SQL (PostgreSQL) with PITR, automated backups, and optimized performance flags.
- **Secure by Default**: Secret Manager integration for DB credentials (no plain-text passwords).
- **Optimized for AI**: Pre-configured for SSE (Server-Sent Events) and large OAuth2 headers.

## 📖 How to Use
1. Copy the content from [SETUP_PROMPT_EN.md](./SETUP_PROMPT_EN.md).
2. Replace `{{GCP_PROJECT_ID}}`, `{{GCP_ACCOUNT}}`, and `{{REGION}}`.
3. Paste it into **Claude Code**, **Cursor**, or any terminal-enabled AI agent.
4. Run the post-setup OAuth2 configuration if you need Google Sheets/Drive integration.
