# Mitsumi Infrastructure docs

Self-hosted platform on **francis-pc (192.168.41.70)**: Coolify for apps and databases, Grafana for metrics and logs,
daily database backups to S3.

| | |
|---|---|
| Coolify (apps, databases, backups) | https://infra.mitsumi.ai |
| Grafana (dashboards, log search) | https://grafana.infra.mitsumi.ai |

Both are reachable **only from the Mitsumi network** (office LAN and Wi-Fi, FortiClient VPN).

## Deploying an app

| Guide | Use it for |
|---|---|
| [deploy-fullstack-react-fastapi-mongo.md](deploy-fullstack-react-fastapi-mongo.md) | React frontend + FastAPI backend + MongoDB, one repo, Docker Compose build pack |
| [deploy-nextjs-docker.md](deploy-nextjs-docker.md) | A single Next.js app, Dockerfile build pack, optional database |
| [port-registry.md](port-registry.md) | Pick a free host port (3400–3499) and record it |

Scripts used by the guides live in `../scripts/` and are copied to the server at `~/mitsumi-infra/scripts/`:
- `add-site.sh <domain> <port> [--internal]`: nginx site + HTTPS certificate for a new app (sudo)

## Operating the platform

| Doc | What's in it |
|---|---|
| [runbook.md](runbook.md) | How everything was installed, backup and restore, migrations done so far, rollback steps, Coolify quirks |
| [coolify-s3-policy.json](coolify-s3-policy.json) | The IAM policy for the backup user (bucket-only) |



testing exposing api key
KEYGUARD_REMOVED_KEYGUARD_KEY
