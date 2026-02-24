# 使い方

1. プロンプト内の3箇所を書き換える：
   - {{GCP_PROJECT_ID}} → GCPプロジェクトID
   - {{GCP_ACCOUNT}} → gcloudログイン済みアカウント
   - {{REGION}} → リージョン（asia-northeast1 等）
2. gcloud auth login を済ませる
3. Claude Code に ## プロンプト本文 以降を貼り付けて実行

---

## 書き換え箇所（3つだけ）

| 変数 | 説明 | 例 |
|------|------|----|
| `{{GCP_PROJECT_ID}}` | GCPプロジェクトID | `tnakajo-n8n-project |
| `{{GCP_ACCOUNT}}` | gcloud認証済みアカウント | `nj.takayuki@gmail.com` |
| `{{REGION}}` | デプロイリージョン | `asia-northeast1` |
