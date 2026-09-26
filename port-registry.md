# Port registry: francis-pc (192.168.41.70)

Every app on this server listens on a **host port**, and nginx forwards its domain to that port.
Two apps can't share a port. Before choosing one, check this list **and** the live server:

```bash
ss -tlnp | grep ':34[0-9][0-9] '     # nothing printed = the 34xx range is free
```

## Rules

- **New apps take the next free port in `3400–3499`.** That range is reserved for Coolify-deployed apps.
- Bind to **`127.0.0.1` only** (`127.0.0.1:3401:3000`). nginx is the only thing that should reach an app.
  A plain `3401:3000` publishes on every interface, which puts the app on the LAN without TLS or the nginx rules.
- Add your app to the table below in the same PR or change that deploys it.

## Reserved for new apps (3400–3499)

| Port | App | Coolify project / resource | Domain |
|---|---|---|---|
| 3400 | *(example, keep free)* | | |

## Already in use (2026-09-25; `ss -tlnp` is the source of truth)

| Port | Bound to | Used by |
|---|---|---|
| 22 | all | SSH |
| 80, 443 | all | host nginx (every `*.mitsumi.ai` site) |
| 3000 | all | EPC tracker |
| 3200 | container | mitsumi-id (auth) |
| 3300 | 127.0.0.1 | Grafana |
| 4000 | 127.0.0.1 | LiteLLM (Coolify: Mitsumi GPT / private-gpt) |
| 4040 | 127.0.0.1 | in use (check `ss -tlnp`) |
| 5000, 8001, 8002 | all | Python APIs (check `ss -tlnp`) |
| 5432 | 127.0.0.1 | Postgres (existing stack) |
| 6001, 6002 | all | Coolify realtime and terminal |
| 6390 | 127.0.0.1 | vexa redis |
| 6443, 10248–10259 | mixed | k3s (Kubernetes) |
| 6789, 9091 | all | metrics exporters |
| 7860 | 127.0.0.1 | mitsumi-gpt web (Coolify: Mitsumi GPT / mitsumi-gpt-web) |
| 7880, 7881 | all | LiveKit (vexa) |
| 7981, 10010, 20180–20182, 20241, 35665, 44201 | 127.0.0.1 | various internal services |
| 8000 | all | Coolify UI (reach it through https://infra.mitsumi.ai) |
| 8025 | all | HP-OSS mailpit |
| 8081 | all | HP-OSS web |
| 8090 | 127.0.0.1 | mitsumi-web (bank recon) |
| 8095, 8096 | 127.0.0.1 | vexa api / web |
| 9000 | all | epc-tracker deploy webhook |
| 9090 | all | legacy Prometheus |
| 11434 | 127.0.0.1 | Ollama (Coolify: Mitsumi GPT / private-gpt) |
| 27017 | 127.0.0.1 | Mongo (existing stack) |
| 27018 | 127.0.0.1 | marketplace-mongo |
