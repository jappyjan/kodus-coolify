# Kodus on Coolify

Docker Compose for self-hosting [Kodus](https://kodus.io) on Coolify.
Secrets are provided via Coolify environment variables (not committed).

Configure these deployment-specific values in Coolify before deploying:

- `KODUS_WEB_IMAGE`
- `KODUS_API_IMAGE`
- `KODUS_WORKER_IMAGE`
- `KODUS_WEBHOOK_IMAGE`
- `NEXTAUTH_URL`
- `API_URL`
- `API_FRONTEND_URL`
- `API_USER_INVITE_BASE_URL`
- `API_GITHUB_CODE_MANAGEMENT_WEBHOOK`
