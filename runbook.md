# Mitsumi Infrastructure — Runbook

Host: `192.168.41.70` (`francis-pc`, Ubuntu 24.04, 8 vCPU, 31 GB RAM, `/home` 152 GB free).
The host already runs production. **Do not touch** nginx's existing sites, k3s, or existing containers.

## Survey (2026-09-24)

| Port | Owner | Notes |
|---|---|---|
| 80 / 443 | host nginx | about 12 sites, certbot TLS. Coolify's Traefik proxy **must stay off** |
| 3000 | EPC app | so Grafana uses **3300** |
| 9090 | legacy Prometheus 2.37 | only scrapes itself. Left alone; we run our own internal one |
| 6443 / 10250 | k3s | untouched |
| 8000 / 6001 / 6002 | *free* | used by Coolify |
| 3300 | *free* | Grafana (loopback only) |

Existing databases: `mitsumi-postgres-1`, `private-gpt-db`, `vexa-postgres-1` (Postgres 16);
`marketplace-mongo` (Mongo 7), `hpossportal-mongo-1` (Mongo 8). These keep running as they are. They can be
brought into Coolify's backups later (see "Importing an existing DB").

`francis` has no passwordless sudo, so the steps marked **(sudo)** must be run by hand.

---

## 1. Install Coolify (sudo)

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | sudo bash
```
Docker is already installed, and the script reuses it. Data goes to `/data/coolify`.

Right after it finishes, **before deploying anything**:
1. Open `http://192.168.41.70:8000` over VPN and create the admin account. Turn on 2FA (Profile > 2FA).
2. **Servers > localhost > Proxy**: choose **None / stop proxy**. This keeps nginx as the owner of 80/443.
3. **Settings > Instance domain**: `https://infra.mitsumi.ai` (after step 2 below).
4. **Keys & Tokens > API tokens**: create a token (read + write, *not* root) for the portal.

Check that production is unharmed:
```bash
docker ps --format '{{.Names}}\t{{.Status}}' | sort
curl -sI https://bank.recon.mitsumi.ai | head -1
```

## 2. Put Coolify behind nginx (sudo)

**Done 2026-09-25.** `https://infra.mitsumi.ai` (Coolify) and `https://grafana.infra.mitsumi.ai` share one certificate, valid until 2026-12-24, which certbot renews.
Coolify > Settings > General > **URL** = `https://infra.mitsumi.ai`. The page saves when you press Enter; there's no Save button.
When testing websockets use `curl --http1.1`, because over HTTP/2 nginx returns a misleading 404.
`francis` needs a password for sudo, so sudo steps are run by a person: `ssh -t ... "sudo bash -c '...'"`.

Create a DNS A record `infra.mitsumi.ai` (and `grafana.infra.mitsumi.ai`) pointing at the same public IP as the other `*.mitsumi.ai` sites.

```bash
sudo cp nginx/infra.mitsumi.ai nginx/grafana.infra.mitsumi.ai /etc/nginx/sites-available/
sudo ln -s /etc/nginx/sites-available/infra.mitsumi.ai /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/grafana.infra.mitsumi.ai /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
sudo certbot --nginx -d infra.mitsumi.ai -d grafana.infra.mitsumi.ai
```

> Docker-published ports (8000, and any DB "public port") skip `ufw`. Limit them to the VPN in the
> `DOCKER-USER` iptables chain, for example:
> `sudo iptables -I DOCKER-USER -p tcp --dport 8000 ! -s <VPN_SUBNET> -j DROP`

## 3. Backups to S3 (off-site)

**Done 2026-09-25.** AWS S3 was used instead of MinIO, since no MinIO exists and S3 is off-site with nothing to run.
- Bucket **`mitsumi-db-backups`** in eu-west-3 (Paris, the account's usual region): all public access blocked, AES-256,
  versioning on (overwritten or deleted backups recoverable for 30 days), incomplete uploads aborted after 7 days.
- IAM user **`coolify-db-backups`** has only the inline policy `only-mitsumi-db-backups` (see `docs/coolify-s3-policy.json`):
  list and read/write/delete on that bucket and nothing else. Its access key went straight into Coolify and is stored nowhere else.
  To rotate it: create a new key in IAM, update it in Coolify > S3 Storages > `aws-s3-eu-west-3`, then delete the old key.
- Coolify S3 storage `aws-s3-eu-west-3` (`tqufsfnx8gihhoq9e2s3miik`), endpoint `https://s3.eu-west-3.amazonaws.com`, validated.
- Every database schedule: daily at 02:00 UTC, 14 kept locally, **30 backups / 90 days in S3**.
  Objects land under `data/coolify/backups/databases/root-team-0/<name>-<uuid>/`.
- Restore drill from S3 on 2026-09-25: the litellm-postgres dump downloaded from S3 restored to 86 tables, 65 spend logs and 161 migrations, identical to live.
- New database: turn on **Backups > S3** (same storage) when adding its schedule.

**Restore drill** (do this once before trusting backups):
1. Provision a new empty DB of the same type in a `restore-test` environment.
2. Take the latest dump from MinIO, or locally from `/data/coolify/backups/databases/root-team-0/<name>-<uuid>/`.
3. Run `pg_restore -d <conn> --no-owner pg-dump-*.dmp` or `mongorestore --uri <conn> --gzip --archive=mongo-dump-all-*.tar.gz`.
   The Mongo file is named `.tar.gz` but is really a gzipped `mongodump --archive`, so plain `tar` fails on it.
4. Compare table and collection counts with the source.

Drill run 2026-09-25 on the Platform Sandbox databases: Postgres 1000/1000 rows and Mongo 500/500 docs restored.

## 4. Provisioning a database

Coolify > Project > Environment > **+ New > Database > PostgreSQL / MongoDB**.
- Leave the **public port** off unless it is needed. Apps on the same host connect over the internal Docker network.
- If a public port is needed, pick one from `5440–5499` (Postgres) or `27030–27099` (Mongo), which avoids the existing 5432/27017/27018. Then restrict it to the VPN as above.
- Add a backup schedule right away (section 3).

### Importing an existing DB
Dump from the old container, provision in Coolify, restore, switch the app's connection string, and keep the old container stopped (not removed) for a week.

### Migrated: mitsumi-gpt / LiteLLM (2026-09-25)
- The old DB was `private-gpt-db` (compose `/home/francis/private-gpt`, service `db`). The new one is Coolify project **Mitsumi GPT**, database `litellm-postgres` (`ak9sodx3xvpdvfj4h4suq6lf`, postgres:16), with daily 02:00 backups and 14 kept.
- Cutover 11:38:58–11:39:41 UTC (43 s of LiteLLM downtime). All 78 tables matched row for row.
- The compose change: `DATABASE_URL` points at the Coolify DB host with `${LITELLM_DB_PASSWORD}` (new key in `.env`), and `litellm` joined the external `coolify` network.
  Backups of the originals are `docker-compose.yml.bak-20260925` and `.env.bak-20260925`. The final pre-cutover dump is `~/mitsumi-infra/litellm-final-20260925.dump`.
- **The old `db` service is still running as a rollback.** It no longer receives writes, so rolling back loses anything written after the cutover.
  - Rollback: `cp docker-compose.yml.bak-20260925 docker-compose.yml && docker compose up -d --no-deps litellm`
  - Retire it after 2026-10-02: remove the `db` service and litellm's `depends_on: db`, then `docker compose up -d --remove-orphans`. Keep `data/postgres` for another month.
- mitsumi-gpt (`/home/francis/mitsumi-gpt`, :7860) is stateless, so the database move didn't touch it.

### Migrated: the rest of mitsumi-gpt (2026-09-25)
The whole project now runs from Coolify project **Mitsumi GPT**:

| Resource | Type | What | Env vars (Coolify UI) |
|---|---|---|---|
| `mitsumi-gpt-web` (`nnxxhb5yxccynjx5l7mgdjd5`) | App, Dockerfile, from `git@github.com:Mitsumi-Distribution/mitsumi-gpt.git` `main` | chat UI on `127.0.0.1:7860` (nginx serves gpt.mitsumi.ai) | `PRIVATE_GPT_URL`, `PRIVATE_GPT_KEY`, `PRIVATE_GPT_MODEL` |
| `private-gpt` (`4h0sy6mqdyewacya1bmgk11a`) | Service, compose in `coolify/mitsumi-gpt/private-gpt.compose.yml` | Ollama `127.0.0.1:11434` + LiteLLM `127.0.0.1:4000` | `LITELLM_MASTER_KEY`, `LITELLM_SALT_KEY`, `LITELLM_DB_PASSWORD` |
| `litellm-postgres` (`ak9sodx3xvpdvfj4h4suq6lf`) | Database | LiteLLM keys and spend | managed by Coolify |

- Cutovers: LiteLLM + Ollama at 11:48:17–11:48:58 UTC (41 s). Web at 12:00:30–12:00:55 (25 s). A later redeploy through Coolify showed about 3 s of downtime.
- The web app reaches LiteLLM as `http://litellm-4h0sy6mqdyewacya1bmgk11a:4000/v1` over the `coolify` network. Both resources have "Connect to predefined network" turned on.
- Ollama still reads the 23 GB of models from `/home/francis/private-gpt/models`, and LiteLLM reads `/home/francis/private-gpt/litellm/config.yaml`. **Don't delete that folder.**
- **Deploying:** push to `main` and it auto-deploys (tested 2026-09-25 with commit 176099d: webhook 200, deploy finished, healthy).
  A repo webhook (id 685566463, push only) calls `https://infra.mitsumi.ai/webhooks/source/github/events/manual`, signed with the
  app's "GitHub webhook secret" (Coolify > app > Webhooks). Manual deploy: Coolify > Deploy, or `POST /api/v1/deploy?uuid=nnxxhb5yxccynjx5l7mgdjd5`.
  Coolify reads the repo with a read-only deploy key ("coolify (francis-pc)").
  Other repos can use the GitHub App `alert-ape-vq2elbjxj9x9lvoy2zwo` (org Mitsumi-Distribution, all repos). Its webhook
  URL must be `https://infra.mitsumi.ai/webhooks/source/github/events`, set in GitHub > org settings > GitHub Apps.
- **Coolify quirks found:**
  - The API rejects IP-bound port mappings (`127.0.0.1:7860:7860`) even though its own pattern allows them. It was set in `coolify-db` directly (`applications.ports_mappings`). Setting it in the UI works too.
  - Never save a plain `7860:7860` mapping, because it would publish on every interface.
  - The app health check needs `curl` in the image.
- **Rollback (until 2026-10-02):**
  - Web: Coolify stop `mitsumi-gpt-web`, then `cd /home/francis/mitsumi-gpt && docker compose start web`.
  - LiteLLM/Ollama: Coolify stop `private-gpt`, then `cd /home/francis/private-gpt && docker compose start ollama litellm`. The old compose still points at the Coolify DB.
- `MOVED-TO-COOLIFY.md` in both old folders warns against `docker compose up` there, and `scripts/deploy.sh` in the repo now refuses to run.

## 5. Monitoring stack

```bash
scp -r monitoring francis@192.168.41.70:~/mitsumi-infra/
ssh francis@192.168.41.70
cd ~/mitsumi-infra/monitoring && cp .env.example .env   # set GRAFANA_ADMIN_PASSWORD
docker compose up -d
```
The `francis` user is in the docker group, so no sudo is needed.
Grafana is at `https://grafana.infra.mitsumi.ai` (or `ssh -L 3300:127.0.0.1:3300` then http://localhost:3300).

Dashboards (1860 and 19792 are already imported, 2026-09-25; the rest go in via Dashboards > New > Import > ID):
- **1860**: Node Exporter Full (host CPU, RAM, disk)
- **19792**: cAdvisor / Docker containers. Needs cAdvisor >= 0.53 because this host's Docker 29 uses the containerd image store
- **9628**: PostgreSQL (after adding a postgres-exporter)
- **2583**: MongoDB (after adding a mongodb-exporter)

Log search: Explore > Loki, for example `{compose_project="hpossportal"} |= "error"` or `{coolify_project="HR"}`.

Alerts: Alerting > Contact points (email/Slack). Suggested rules:
- disk `/` or `/home` over 85% (`node_filesystem_avail_bytes`)
- a DB container down (`absent(container_last_seen{name=~".*postgres.*"})`)
- Loki `{container=~"coolify.*"} |= "backup" |= "failed"`
