# Deploy a single Next.js app on Coolify (Dockerfile)

This guide takes a **Next.js** repo (App Router, Next 14/15) to `https://<app>.mitsumi.ai`, built from a
Dockerfile by Coolify and redeployed on every push to `main`.

Worked example on this server: **mitsumi-gpt** (`Mitsumi-Distribution/mitsumi-gpt`, Coolify project *Mitsumi GPT*)
uses the same Dockerfile-app pattern (Python instead of Node).
For an app with a separate API and database, see [deploy-fullstack-react-fastapi-mongo.md](deploy-fullstack-react-fastapi-mongo.md).

---

## How it fits together

```
 browser ──https──▶ host nginx (443, certbot TLS) ──▶ 127.0.0.1:34xx ──▶ Coolify app (Next.js standalone) :3000
                                                                      │  (optional) coolify network
                                                                      └──▶ Coolify Postgres / Mongo, backed up to S3
```

### Mitsumi specifics (read once)

1. **Coolify's proxy is OFF.** nginx on the host owns ports 80/443. Leave the Coolify **Domains** field **empty**,
   publish a `127.0.0.1` port, and add an nginx site (step 5).
2. **Port mapping must start with `127.0.0.1:`**, e.g. `127.0.0.1:3402:3000`. Set it **in the Coolify UI**:
   the Coolify *API* wrongly rejects IP-bound mappings. Never save a plain `3402:3000`, because it publishes the app on
   every interface. Take the port from [port-registry.md](port-registry.md) (range `3400–3499`).
3. **Health checks need `curl` or `wget` in the image.** `node:*-alpine` has `wget`, so the Dockerfile below is fine as-is.
4. **`NEXT_PUBLIC_*` values are baked in at BUILD time.** Mark them *Build time* in Coolify, and redeploy (not restart) after changing one.
   **Never put secrets in `NEXT_PUBLIC_*`**: they end up in the JavaScript every visitor downloads.
5. **Using a Coolify database?** Turn on **Connect To Predefined Network** on the app, or the DB hostname won't resolve.

---

## 1. Prepare the repo

**`next.config.ts`** (or `.js`): standalone output makes a small, self-contained server.
```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  output: "standalone",
  poweredByHeader: false,
};

export default nextConfig;
```

**`app/api/health/route.ts`**: what Coolify's health check calls.
```ts
export const dynamic = "force-dynamic";   // never cached, always a live check

export function GET() {
  return Response.json({ ok: true });
}
```
If the app has a database, check it here too (run a trivial query), so "healthy" means the app actually works.

**`Dockerfile`**
```dockerfile
# ---------- deps ----------
FROM node:22-alpine AS deps
WORKDIR /app
RUN apk add --no-cache libc6-compat && corepack enable
COPY package.json pnpm-lock.yaml* package-lock.json* yarn.lock* ./
RUN if [ -f pnpm-lock.yaml ]; then pnpm install --frozen-lockfile; \
    elif [ -f yarn.lock ]; then yarn install --frozen-lockfile; \
    else npm ci; fi

# ---------- build ----------
FROM node:22-alpine AS build
WORKDIR /app
RUN corepack enable
COPY --from=deps /app/node_modules ./node_modules
COPY . .
# Public values are inlined into the client bundle at build time.
# Declare every NEXT_PUBLIC_* you use as an ARG; Coolify passes build-time vars as build args.
ARG NEXT_PUBLIC_APP_NAME="Mitsumi App"
ARG NEXT_PUBLIC_API_BASE=""
ENV NEXT_PUBLIC_APP_NAME=$NEXT_PUBLIC_APP_NAME \
    NEXT_PUBLIC_API_BASE=$NEXT_PUBLIC_API_BASE \
    NEXT_TELEMETRY_DISABLED=1
RUN if [ -f pnpm-lock.yaml ]; then pnpm build; \
    elif [ -f yarn.lock ]; then yarn build; \
    else npm run build; fi

# ---------- run ----------
FROM node:22-alpine AS run
WORKDIR /app
ENV NODE_ENV=production \
    NEXT_TELEMETRY_DISABLED=1 \
    PORT=3000 \
    HOSTNAME=0.0.0.0
RUN addgroup -S -g 1001 nodejs && adduser -S -u 1001 -G nodejs nextjs
COPY --from=build --chown=nextjs:nodejs /app/public ./public
COPY --from=build --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=build --chown=nextjs:nodejs /app/.next/static ./.next/static
USER nextjs
EXPOSE 3000
# wget ships with alpine, which Coolify's health check needs.
HEALTHCHECK --interval=15s --timeout=5s --start-period=20s --retries=5 \
  CMD wget -qO- http://localhost:3000/api/health || exit 1
CMD ["node", "server.js"]
```
If the project has no `public/` folder, create an empty one (`public/.gitkeep`), or remove that `COPY` line.

**`.dockerignore`**
```
node_modules
.next
.git
.env*
npm-debug.log*
Dockerfile
.dockerignore
```

**Test locally before pushing:**
```bash
docker build -t myapp --build-arg NEXT_PUBLIC_APP_NAME="Orders" .
docker run --rm -p 3000:3000 -e DATABASE_URL=... myapp
curl -s localhost:3000/api/health     # {"ok":true}
```

Push the repo to **Mitsumi-Distribution** on GitHub. The Coolify GitHub App (`alert-ape-…`) can see every repo in the org.

---

## 2. (Optional) Database

If the app needs a database, create it **in Coolify first** so it gets backups:
1. Project → **+ New → Databases → PostgreSQL** (or MongoDB). Image `postgres:16`. **"Make it publicly available" OFF.**
2. **Start** it, then copy **"Postgres URL (internal)"**. That becomes the app's `DATABASE_URL`.
3. **Backups → + Add**: `0 2 * * *`, **Save to S3 ON** with storage `aws-s3-eu-west-3`, keep 14 locally and 30 backups / 90 days in S3.
   Click **Backup Now** once and check that it succeeded and uploaded.

## 3. Create the app in Coolify

1. **Projects → + Add** (e.g. `Orders`; no `:` in the description), or open an existing project → environment `production`.
2. **+ New → Private Repository (with GitHub App)** → **`alert-ape-…`** → pick the repo → branch **`main`**.
3. **Build Pack: Dockerfile**. Dockerfile location `/Dockerfile`, base directory `/` (or the subfolder, in a monorepo). Continue.
4. **General**:
   - **Domains: EMPTY.** Delete any `sslip.io` domain Coolify pre-filled.
   - **Ports Exposes:** `3000`
   - **Ports Mappings:** `127.0.0.1:3402:3000` (your port, with the `127.0.0.1:` prefix)
5. **Health Check**: enabled, path `/api/health`, port `3000`.
6. **Resource Limits**: memory `512m` (raise it for heavy SSR), CPU `1`.
7. **Advanced**: turn on **Connect To Predefined Network** *only if* the app uses a Coolify database (step 2).
8. **Environment Variables**:

   | Key | Example | Build time | Runtime | Notes |
   |---|---|---|---|---|
   | `NEXT_PUBLIC_APP_NAME` | `Orders` | ☑ | ☐ | public, baked into the bundle |
   | `NEXT_PUBLIC_API_BASE` | `https://orders.mitsumi.ai` | ☑ | ☐ | public |
   | `DATABASE_URL` | internal DB URL from step 2 | ☐ | ☑ | secret, server only |
   | `AUTH_SECRET` / API keys | … | ☐ | ☑ | secret, server only |

   Server-side code (route handlers, server components, server actions) reads runtime variables through `process.env` on every request.
   Client components only see `NEXT_PUBLIC_*`, and only the values from the last **build**.
9. **Deploy**, and watch the build log. Then on the server:
   ```bash
   curl -s http://127.0.0.1:3402/api/health     # {"ok":true}
   ss -tlnp | grep 3402                          # must show 127.0.0.1:3402
   ```
10. **Auto-deploy:** check that GitHub → Mitsumi-Distribution → Settings → Developer settings → GitHub Apps →
    `alert-ape-…` → **Webhook URL** is `https://infra.mitsumi.ai/webhooks/source/github/events`.
    Then push a small commit and watch a deployment start by itself.

## 4. DNS

The `mitsumi.ai` zone is in Route 53 (AWS CLI on an admin PC):
```bash
aws route53 change-resource-record-sets --hosted-zone-id Z07902582CM9FUW7TVZ5O --change-batch '{
  "Changes":[{"Action":"CREATE","ResourceRecordSet":{"Name":"orders.mitsumi.ai","Type":"A","TTL":300,
  "ResourceRecords":[{"Value":"102.210.173.202"}]}}]}'
nslookup orders.mitsumi.ai 8.8.8.8     # wait for 102.210.173.202
```

## 5. nginx and HTTPS

On the server (asks for your sudo password):
```bash
sudo bash ~/mitsumi-infra/scripts/add-site.sh orders.mitsumi.ai 3402
# staff-only tool? add --internal to allow only the Mitsumi network:
# sudo bash ~/mitsumi-infra/scripts/add-site.sh orders.mitsumi.ai 3402 --internal
```
It creates the site, tests the nginx config before reloading (and backs out if the test fails), then gets a
Let's Encrypt certificate with an http→https redirect. Existing sites are never touched.

Add the port to [port-registry.md](port-registry.md).

---

## 6. Check it's done

- [ ] `https://orders.mitsumi.ai` loads with a valid certificate, and `http://` redirects
- [ ] `https://orders.mitsumi.ai/api/health` → `{"ok":true}`
- [ ] Coolify shows the app **Running (healthy)**
- [ ] A commit to `main` deployed by itself
- [ ] `ss -tlnp` shows `127.0.0.1:3402`, not `0.0.0.0:3402`
- [ ] If there's a database: a successful backup **uploaded to S3**
- [ ] Port recorded in `port-registry.md`

## Day-2 operations

| Task | Where |
|---|---|
| Live logs | Coolify → app → **Logs** |
| Search old logs | https://grafana.infra.mitsumi.ai → Explore → Loki → `{coolify_project="orders"}` |
| CPU / memory | Grafana → *cadvisor dashboard* |
| Change a runtime variable | Coolify → Environment Variables → **Restart** |
| Change a `NEXT_PUBLIC_*` | Coolify → Environment Variables → **Redeploy** |
| Roll back | Coolify → app → **Rollback** → pick a previous commit's image |
| Open a shell | Coolify → app → **Terminal** |

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Build fails: `.next/standalone` not found | `output: "standalone"` missing | Add it to `next.config` |
| Page loads without CSS or JS (404s on `/_next/static`) | `.next/static` not copied | Keep the `COPY … .next/static` line |
| Container exits: `EADDRINUSE` or the app isn't reachable | Bound to localhost inside the container | `HOSTNAME=0.0.0.0` (in the Dockerfile) |
| App **unhealthy** | Health route missing, or the image has no curl/wget | Add `app/api/health/route.ts`; keep an alpine base |
| New `NEXT_PUBLIC_*` value not showing | Build-time variable | **Redeploy** |
| `Can't reach database server` | Not on the `coolify` network | **Connect To Predefined Network**, then redeploy |
| Deploy fails: *port is already allocated* | Port in use | Pick another: `ss -tlnp`, [port-registry.md](port-registry.md) |
| The Ports Mappings field won't save `127.0.0.1:…` | Coolify validation bug | Ask the infra admin; the workaround is in [runbook.md](runbook.md) ("Coolify quirks") |
| Push didn't deploy | GitHub App webhook URL | See step 3.10; GitHub → App → Advanced → Recent deliveries |
| **403** in the browser | Site was added with `--internal` | Connect FortiClient VPN |
