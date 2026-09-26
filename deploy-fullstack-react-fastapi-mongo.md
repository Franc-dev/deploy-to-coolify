# Deploy a full-stack app on Coolify: React + FastAPI + MongoDB

This guide takes a repo with a **React** (Vite) frontend, a **FastAPI** backend and a **MongoDB**
database from nothing to `https://<app>.mitsumi.ai`, deployed by Coolify on every push to `main`.

It follows how this server is set up. Some steps differ from the Coolify docs, so don't skip the **Mitsumi specifics**.

| | |
|---|---|
| Coolify | https://infra.mitsumi.ai (Mitsumi network only: office LAN, Wi-Fi, FortiClient VPN) |
| Server | francis-pc, 192.168.41.70, public IP 102.210.173.202 |
| Worked example | HP-OSS (`Mitsumi-Distribution/HP-OSS`) uses the same frontend/backend/Mongo shape |

---

## How it fits together

```
 browser ──https──▶ host nginx (443, certbot TLS) ──▶ 127.0.0.1:34xx
                                                         │
                          Coolify app (Docker Compose build pack, one repo)
                          ┌──────────────────────────────┴───────────────┐
                          │ web (nginx + React build)  :80               │
                          │   /        → static React files              │
                          │   /api/*   → backend:8000 ─┐                 │
                          │ backend (FastAPI/uvicorn)  :8000             │
                          └────────────────────────────┼─────────────────┘
                                                       │ coolify network
                          Coolify database (MongoDB)  <uuid>:27017
                          daily backups → server disk + S3 (mitsumi-db-backups)
```

**Why this shape:**
- **One domain, no CORS.** The browser only talks to `web`, and `web` proxies `/api` to the backend.
- **Only one port is published** (`web`), on `127.0.0.1`. The backend and Mongo are never exposed.
- **Mongo is a Coolify database, not a service inside your compose.** Coolify gives it daily backups to
  S3, restore, logs and a status page. A Mongo container inside your own compose gets none of that
  (HP-OSS ran for 3 weeks with broken backups because of it).

### Mitsumi specifics (read once)

1. **Coolify's proxy is OFF.** nginx on the host owns ports 80/443. Coolify does **not** route domains:
   leave the Coolify "Domains" field **empty**, publish a `127.0.0.1` port, and add an nginx site (step 6).
2. **Port mappings must start with `127.0.0.1:`.** A plain `3401:80` puts the app on the LAN without TLS.
   Take the next free port from [port-registry.md](port-registry.md) (range `3400–3499`).
3. **Health checks need `curl` or `wget` inside the image.** `python:*-slim` has neither, so the Dockerfile below installs curl.
   Without it Coolify marks the app **unhealthy** and rolls back later deploys.
4. **To reach a Coolify database, turn on "Connect to Predefined Network"** on the app. Otherwise the DB hostname won't resolve.
5. **nginx inside `web` must re-resolve the backend.** Backend containers get a new IP on every deploy or restart.
   A plain `proxy_pass http://backend:8000` looks the name up once and then sends traffic to a dead IP.
   HP-OSS returned 502 for a day because of this. The nginx.conf below uses Docker's resolver so it always finds the current IP.
6. **Coolify descriptions can't contain `:`**, because the API rejects them.
7. **sudo** on the server needs your password, so the nginx/certbot step is run by a person (step 6).

---

## 1. Repo layout

```
my-app/
├── backend/
│   ├── app/
│   │   ├── __init__.py
│   │   └── main.py
│   ├── requirements.txt
│   └── Dockerfile
├── frontend/
│   ├── src/…            (Vite + React)
│   ├── nginx.conf
│   ├── package.json
│   └── Dockerfile
├── docker-compose.yml          # local development
├── docker-compose.coolify.yml  # what Coolify deploys
├── .dockerignore               # in backend/ and frontend/ too
└── .env.example                # names only, never real values
```

## 2. Backend (FastAPI)

**`backend/app/main.py`**: read every setting from the environment, and give Coolify a health endpoint.
```python
import os
from contextlib import asynccontextmanager

from fastapi import APIRouter, FastAPI
from motor.motor_asyncio import AsyncIOMotorClient

MONGODB_URL = os.environ["MONGODB_URL"]          # fail fast if it's missing
MONGODB_DB = os.environ.get("MONGODB_DB", "app")


@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.mongo = AsyncIOMotorClient(MONGODB_URL, serverSelectionTimeoutMS=5000)
    app.state.db = app.state.mongo[MONGODB_DB]
    yield
    app.state.mongo.close()


app = FastAPI(lifespan=lifespan, docs_url="/api/docs", openapi_url="/api/openapi.json")

# Every route lives under /api: the web container forwards /api/* here unchanged.
api = APIRouter(prefix="/api")


@api.get("/health")
async def health():
    await app.state.db.command("ping")          # 500 if Mongo is unreachable
    return {"ok": True}


@api.get("/items")
async def items():
    return await app.state.db.items.find({}, {"_id": 0}).to_list(100)


app.include_router(api)
```

**`backend/requirements.txt`**
```
fastapi>=0.115
uvicorn[standard]>=0.30
motor>=3.5
```

**`backend/Dockerfile`**
```dockerfile
FROM python:3.12-slim

# curl: Coolify's health check runs it inside the container.
RUN apt-get update && apt-get install -y --no-install-recommends curl \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app ./app

RUN useradd --create-home --uid 10001 appuser
USER appuser

ENV PYTHONUNBUFFERED=1
EXPOSE 8000
HEALTHCHECK --interval=15s --timeout=5s --retries=5 \
  CMD curl -fsS http://localhost:8000/api/health || exit 1
# 0.0.0.0 inside the container. It is not published to the host; only `web` reaches it.
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--proxy-headers", "--forwarded-allow-ips", "*"]
```

**`backend/.dockerignore`**
```
.venv/
__pycache__/
*.pyc
.env
tests/
```

## 3. Frontend (React + nginx)

Call the API with **relative URLs** (`fetch("/api/items")`) so the same build works locally and in production.

**`frontend/nginx.conf`**
```nginx
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;

    # Docker's embedded DNS. Re-resolve `backend` every 10s so a redeployed backend
    # (new IP) is found. Without this, nginx keeps the old IP and returns 502.
    resolver 127.0.0.11 valid=10s ipv6=off;
    set $backend http://backend:8000;

    location /api/ {
        proxy_pass $backend;                 # a variable forces a fresh lookup
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $http_x_forwarded_proto;
        proxy_read_timeout 120s;
    }

    location = /health { return 200 "ok\n"; }

    # index.html must never be cached, or users keep an old app after a deploy.
    location = /index.html { add_header Cache-Control "no-cache"; }
    # Hashed build assets can be cached forever.
    location /assets/ { add_header Cache-Control "public, max-age=31536000, immutable"; }

    # React Router: unknown paths serve the app.
    location / { try_files $uri $uri/ /index.html; }
}
```

**`frontend/Dockerfile`**
```dockerfile
# ---- build ----
FROM node:22-alpine AS build
WORKDIR /app
RUN corepack enable
COPY package.json pnpm-lock.yaml* package-lock.json* ./
RUN if [ -f pnpm-lock.yaml ]; then pnpm install --frozen-lockfile; else npm ci; fi
COPY . .
# VITE_* values are baked into the JS at BUILD time (see step 5, "Build time").
ARG VITE_APP_NAME="Mitsumi App"
ENV VITE_APP_NAME=$VITE_APP_NAME
RUN if [ -f pnpm-lock.yaml ]; then pnpm build; else npm run build; fi

# ---- serve ----
FROM nginx:1.27-alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
# wget ships with alpine, so Coolify's health check works.
HEALTHCHECK --interval=15s --timeout=5s --retries=5 \
  CMD wget -qO- http://localhost/health || exit 1
```

**`frontend/.dockerignore`**
```
node_modules/
dist/
.env*
```

## 4. `docker-compose.coolify.yml`

```yaml
# Deployed by Coolify (Docker Compose build pack). Mongo is NOT here: it's a Coolify
# database, reached over the `coolify` network via "Connect to Predefined Network".
services:
  backend:
    build: ./backend
    restart: unless-stopped
    environment:
      MONGODB_URL: ${MONGODB_URL}        # set in Coolify > Environment Variables
      MONGODB_DB: ${MONGODB_DB:-app}
    mem_limit: 512m
    cpus: 1

  web:
    build:
      context: ./frontend
      args:
        VITE_APP_NAME: ${VITE_APP_NAME:-Mitsumi App}
    restart: unless-stopped
    depends_on:
      backend:
        condition: service_healthy
    ports:
      - "127.0.0.1:3401:80"               # your port from docs/port-registry.md, loopback only
    mem_limit: 256m
    cpus: 0.5
```

For local development, `docker-compose.yml` can add its own `mongo:8.0` service and `MONGODB_URL: mongodb://mongo:27017`.
Keep that file separate from the Coolify one.

Commit and push everything to a repo in **Mitsumi-Distribution**. The Coolify GitHub App (`alert-ape-…`) can see every repo in the org.

---

## 5. Coolify setup

### 5a. Project
**Projects → + Add**. Use the app's name (e.g. `Orders`), and keep `:` out of the description. It comes with a `production` environment.

### 5b. MongoDB (do this first)
1. In the project: **+ New → Databases → MongoDB**.
2. **Image** `mongo:8.0`, **Name** `<app>-mongo`. **Leave "Make it publicly available" OFF.**
3. **Resource limits → Memory** `1g`.
4. Click **Start** and wait until it's *Running (healthy)*.
5. Copy **"Mongo URL (internal)"**. It looks like `mongodb://root:<password>@<uuid>:27017/?directConnection=true`.
   That's your `MONGODB_URL`.
6. **Backups → + Add**:
   - Frequency `0 2 * * *` (daily 02:00 UTC)
   - **Save to S3: ON**, storage **`aws-s3-eu-west-3`**
   - Keep locally `14`, keep in S3 `30` backups / `90` days
   - Click **Backup Now** once and check the run says *success* and *uploaded to S3*.

### 5c. The app
1. In the project: **+ New → Private Repository (with GitHub App)** → app **`alert-ape-…`** → pick the repo → branch `main`.
2. **Build Pack: Docker Compose**. **Docker Compose Location:** `/docker-compose.coolify.yml`. Continue.
3. **General**:
   - **Domains for each service: leave EMPTY.** Coolify's proxy is off; nginx does the domains (step 6).
     If Coolify pre-filled an `sslip.io` domain, delete it.
4. **Advanced → turn ON "Connect To Predefined Network"**, so `backend` can resolve the Mongo `<uuid>` hostname.
5. **Environment Variables**. Coolify lists every `${VAR}` it found in the compose file:

   | Key | Value | Build time | Runtime |
   |---|---|---|---|
   | `MONGODB_URL` | internal Mongo URL from 5b | ☐ | ☑ |
   | `MONGODB_DB` | `orders` | ☐ | ☑ |
   | `VITE_APP_NAME` | `Orders` | ☑ | ☐ |

   **Build time:** anything the React build reads (`VITE_*`) is **baked into the JavaScript** at build.
   Changing it needs a **redeploy**, not just a restart. **Never put secrets in `VITE_*`**, because every visitor can read them.
6. **Deploy**. Watch the build log. When it finishes, check on the server that it's listening:
   ```bash
   curl -s http://127.0.0.1:3401/health        # ok
   curl -s http://127.0.0.1:3401/api/health    # {"ok":true}  (backend + Mongo work)
   ```
7. **Auto-deploy on push:** Coolify apps created with the GitHub App deploy on push automatically, *if* the
   GitHub App's webhook URL is `https://infra.mitsumi.ai/webhooks/source/github/events`
   (GitHub → Mitsumi-Distribution → Settings → Developer settings → GitHub Apps → `alert-ape-…`).
   Check it once. nginx already lets GitHub's IP ranges reach `/webhooks/`.

---

## 6. Domain and HTTPS

**DNS** (from a PC with the AWS CLI; the `mitsumi.ai` zone is in Route 53):
```bash
aws route53 change-resource-record-sets --hosted-zone-id Z07902582CM9FUW7TVZ5O --change-batch '{
  "Changes":[{"Action":"CREATE","ResourceRecordSet":{"Name":"orders.mitsumi.ai","Type":"A","TTL":300,
  "ResourceRecords":[{"Value":"102.210.173.202"}]}}]}'
nslookup orders.mitsumi.ai 8.8.8.8      # wait until it returns 102.210.173.202
```

**nginx + certificate** (on the server; asks for your sudo password):
```bash
sudo bash ~/mitsumi-infra/scripts/add-site.sh orders.mitsumi.ai 3401
# add --internal to limit it to the Mitsumi network, like Coolify/Grafana:
# sudo bash ~/mitsumi-infra/scripts/add-site.sh orders.mitsumi.ai 3401 --internal
```
The script refuses to overwrite an existing site, reloads nginx only if the config test passes, then gets the certificate.

**Record the port** in [port-registry.md](port-registry.md).

---

## 7. Check it's done

- [ ] `https://orders.mitsumi.ai` loads, the padlock is valid, and http redirects to https
- [ ] `https://orders.mitsumi.ai/api/health` → `{"ok":true}`
- [ ] Coolify shows the app and the Mongo DB **Running (healthy)**
- [ ] Mongo has a backup that ran **successfully and uploaded to S3**
- [ ] A test commit to `main` triggered a deployment by itself
- [ ] `ss -tlnp | grep 3401` shows **`127.0.0.1:3401`**, not `0.0.0.0`
- [ ] The port is in `port-registry.md`

## Day-2 operations

| Task | Where |
|---|---|
| Logs (live) | Coolify → app → **Logs** |
| Logs (search, history) | https://grafana.infra.mitsumi.ai → Explore → Loki: `{coolify_project="orders"}`, or narrow with `coolify_resource="<app name>"` |
| CPU / memory | Grafana → *cadvisor dashboard* |
| Change a setting | Coolify → Environment Variables, then **Restart** (runtime) or **Redeploy** (build time) |
| Roll back a bad deploy | Coolify → app → **Deployments / Rollback** → pick the previous commit |
| Restore the database | [runbook.md](runbook.md), section 3 (drill with `mongorestore --gzip --archive`) |
| Shell into a container | Coolify → app → **Terminal** |

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Deploy "finished" but the app shows **unhealthy** | No `curl`/`wget` in the image | Install curl (backend Dockerfile above) |
| `/api/*` returns **502** after a backend redeploy | Frontend nginx cached the old backend IP | Use the `resolver` + `set $backend` pattern above |
| Backend logs `ServerSelectionTimeoutError` / name not resolved | App isn't on the `coolify` network | Turn on **Connect To Predefined Network**, then redeploy |
| Deploy fails: *port is already allocated* | Port taken | Pick a free one: `ss -tlnp`, [port-registry.md](port-registry.md) |
| API returns 422 / "Validation failed" on description | `:` in a Coolify description | Remove it |
| Push didn't deploy | GitHub App webhook URL wrong | See 5c step 7; check GitHub → App → Advanced → Recent deliveries |
| Site shows **403** | `--internal` site and you're outside the Mitsumi network | Connect FortiClient VPN |
| Changed `VITE_*` but nothing changed | It's build-time | **Redeploy** |
