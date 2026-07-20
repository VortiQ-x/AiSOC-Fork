# AiSOC on Dokploy — VortiQ-x Internal Developer Guide

> **Audience:** VortiQ-x engineers working on our AiSOC fork.
> **Scope:** the development deployment on the internal Dokploy VM, reachable at
> `https://ai-soc.vortiqx.com`. This is a **dev environment** — auth bypass is on
> and writes are open. It is **not** a production install.
>
> **Source of truth for the stack:** [`docker-compose.dokploy.yml`](../../docker-compose.dokploy.yml) at the repo root.

---

## 1. At a glance

| | |
|---|---|
| **Repo** | `git@github.com:VortiQ-x/AiSOC-Fork.git` (`origin`) — upstream is `beenuar/AiSOC` |
| **Deployed branch** | `development` |
| **Compose file** | `docker-compose.dokploy.yml` (repo root) |
| **Dokploy service** | "Fullstack AISOC" (`aisoc-fullstack-aisoc-atdak2`) |
| **URL** | `https://ai-soc.vortiqx.com` (Cloudflare Tunnel → Dokploy Traefik → `web:3000`) |
| **App version** | AiSOC v7.6.0 (MIT) |
| **Mode** | DEV — build-from-source, auth bypass on, **writes enabled** |
| **LLM** | on-prem LiteLLM router `http://192.168.20.12:4000/v1` (see the vLLM box docs) |
| **Postgres (direct)** | `<VM-IP>:55432` (host port; container port is 5432) |

**The developer promise of this setup:** edit code in the fork → `git push` →
Dokploy rebuilds the changed service → your change is live. Plus you can write
through the UI/API and inject rows straight into Postgres.

---

## 2. What is deployed

AiSOC is an open-source AI SOC: it ingests security events, normalizes them to
OCSF, enriches + correlates them, runs an LLM agent investigation, and shows the
result in a console. Our Dokploy stack runs the **full pipeline minus
monitoring/extras** — 20 containers in three groups.

### 2.1 Datastores / infra (pulled images — you don't edit these)

| Service | Image | Purpose |
|---|---|---|
| `postgres` | `postgres:16-alpine` | Primary DB: cases, alerts, users, tenants, connectors. **Published on `55432`** for direct injection. Schema SQL mounted at `services/api/migrations`. |
| `redis` | `redis:7-alpine` | Cache, queues, pub/sub between services. |
| `zookeeper` | `cp-zookeeper:7.5.0` | Kafka coordinator (leave alone). |
| `kafka` | `cp-kafka:7.5.0` | Event bus — backbone of ingest→enrich→fuse→agents. |
| `clickhouse` | `clickhouse-server:23.8` | Analytics event lake. Backs **Explore / Lake**. |
| `opensearch` | `opensearch:2.11.0` | Log search engine. Backs **Hunt**. Needs host `vm.max_map_count>=262144`. |
| `qdrant` | `qdrant:v1.7.0` | Vector DB — agent RAG / memory. |
| `neo4j` | `neo4j:5.15-community` | Graph DB — entity relationships. Backs **Attack Graph**. |

### 2.2 Application services (BUILT FROM OUR FORK — this is where you code)

Each has `build:` + `pull_policy: build`, so a redeploy rebuilds it **from our
source**; the upstream `ghcr.io/beenuar/*` images are never pulled.

| Service | Lang | Build context | What it does |
|---|---|---|---|
| `api` | Python / FastAPI | `services/api` | **Backend core.** All `/api/v1/*`, auth, RBAC, DB migrations, fusion gateway. Most-edited service. |
| `agents` | Python / LangGraph | `services/agents` | **AI investigator (~600 lines).** Auto-triage, investigation, Investigation Ledger. Calls our LiteLLM. |
| `web` | Next.js | repo root + `apps/web/Dockerfile` | **SOC console** (~70 pages). Built with demo mode OFF so writes are usable. |
| `ingest-worker` | Go | `services/ingest` | Raw telemetry → OCSF normalize → entity graph into Neo4j. |
| `enrichment` | Go | `services/enrichment` | Enrich alerts with VirusTotal / AbuseIPDB / Shodan / GreyNoise. |
| `fusion` | Python | `services/fusion` | Correlation + Risk-Based Alerting — entity-risk scores, ML ranking. |
| `actions` | Python | `services/actions` | Response/playbook execution (isolate host, disable user…). |
| `connectors` | Python | `services/connectors` | Integration runtime for 78 vendors (CrowdStrike, AWS…). |
| `threatintel` | Python | `services/threatintel` | Threat-intel feeds (MISP, OTX, TAXII/STIX). Also reads the LLM env. |
| `ueba` | Python | `services/ueba` | User & Entity Behavior Analytics — anomaly detection. |
| `realtime` | Node.js | `services/realtime` | WebSocket push to the console (live updates). |

### 2.3 One-shot

| Service | Purpose |
|---|---|
| `seed` | Runs `python -m app.scripts.seed_demo` once, then exits 0. Creates the **admin user** `demo@tryaisoc.com` + 15 demo cases + 28 alerts. Idempotent. Keep it — it creates the user the auth bypass resolves to. |

### 2.4 Data flow

```
connectors / ingest-worker  ─┐
                             ├─► kafka ─► enrichment ─► fusion ─► agents ─► api ─► web
osquery / external telemetry ┘                        (correlate)  (AI)         (UI)
                                                                     │
                                        realtime ◄────── live push ──┘
                                        actions  ◄────── response dispatch
```

### 2.5 What is intentionally NOT deployed

`kafka-ui`, `prometheus`, `grafana`, `alertmanager`, `honeytokens`,
`purple-team`, `osquery-tls`, `slack-bot`, `demo-producer`. Add them later from
the upstream `docker-compose.yml` if needed.

---

## 3. The dev configuration — why writes work

Two **independent** switches control behavior (both verified in source):

| Switch | Where | Value here | Effect |
|---|---|---|---|
| `ENV` / `ENVIRONMENT` | backend env | `development` | **Auth bypass ON.** A token-less request resolves to the seeded admin `demo@tryaisoc.com`. Also enables schema auto-bootstrap on boot (§7.3). |
| `AISOC_DEMO_MODE` | backend env | *unset (false)* | **Writes ENABLED.** `DemoModeMiddleware` would reject mutations if `true`. |
| `NEXT_PUBLIC_DEMO_MODE` | web **build arg** | `""` | UI shows write buttons, no demo banner, no forced auto-login. Baked at build time. |

So: you land as **admin**, with **no login step**, and can **create / update /
delete** for real. (The seeded user `demo@tryaisoc.com` / `aisoc-demo` also works
at `/login` if `ENV` is ever flipped to `production`.)

> ⚠️ If you ever see the "All write actions are disabled" banner on
> `ai-soc.vortiqx.com`, the `web` image was built with demo mode on — rebuild with
> `NEXT_PUBLIC_DEMO_MODE=""` (it's already set in the compose `web.build.args`).

---

## 4. Environment variables (set in Dokploy → Environment)

**Required** (deploy fails loudly if missing — `${VAR:?}` guards):

| Var | Notes |
|---|---|
| `POSTGRES_PASSWORD` | strong random |
| `REDIS_PASSWORD` | strong random |
| `CLICKHOUSE_PASSWORD` | strong random |
| `NEO4J_PASSWORD` | strong random, **min 8 chars** |
| `SECRET_KEY` | strong random, **≥32 chars** — signs API tokens |
| `LLM_BASE_URL` | `http://192.168.20.12:4000/v1` (LiteLLM) |
| `LLM_API_KEY` | a LiteLLM **virtual key**, not the master key |

**Optional:**

| Var | Default | Notes |
|---|---|---|
| `LLM_MODEL` | `qwen2.5-32b` | any model the LiteLLM router serves |
| `AISOC_PUBLIC_URL` | *(empty)* | set to `https://ai-soc.vortiqx.com` — added to API CORS origins |
| `VIRUSTOTAL_API_KEY`, `ABUSEIPDB_API_KEY`, `SHODAN_API_KEY`, `GREYNOISE_API_KEY` | *(empty)* | enrichment providers |
| `MISP_URL`, `OTX_API_KEY`, `TAXII_*` | *(empty)* | threat-intel feeds |

> The LLM is wired with `AISOC_AIRGAPPED=true` on `api`, `agents`, and
> `threatintel`. That guard blocks egress to `api.openai.com` and refuses an
> empty base URL — it closes the silent "fall back to OpenAI cloud" path if the
> local endpoint is unreachable. Keep it on.

---

## 5. How to access it

- **Console:** `https://ai-soc.vortiqx.com` (Cloudflare Tunnel → Traefik → `web:3000`).
  Domain routing is applied by **redeploying** the compose after any domain change.
- **Postgres:** `psql "postgresql://aisoc:<POSTGRES_PASSWORD>@<VM-IP>:55432/aisoc"`
  (or DBeaver with the same creds). Host port is **55432** because the VM already
  runs other Postgres on 5432.
- **API (from inside the stack):** `http://api:8000`. Not published externally;
  the browser reaches it same-origin via the Next.js proxy (`/api/v1/...`).
- **Logs:** Dokploy → service → **Logs** tab, pick a container. Or on the VM:
  `docker logs <container> --tail 100 -f`.
- **Shell into a container:** Dokploy → **Containers** → terminal, or
  `docker exec -it <container> sh`.

---

## 6. What you can modify — the developer map

| You want to change… | Edit here | Rebuilds |
|---|---|---|
| A REST endpoint / API behavior | `services/api/app/api/v1/endpoints/*.py` (+ `router.py`) | `api` |
| A DB model | `services/api/app/models/*.py` | `api` (`create_all` picks it up in dev) |
| DB schema (raw SQL) | new file in `services/api/migrations/NNN_*.sql` | `api` (auto-applied on boot, §7.3) |
| The AI agent / investigation logic | `services/agents/app/` (graph in `app/`, LLM in `app/security/llm_resolver.py`) | `agents` |
| Seeded/demo data | `services/api/app/scripts/seed_demo.py` | `seed` (+`api`, same image) |
| The console UI | `apps/web/src/app/**` (pages), `apps/web/src/lib/api.ts` (API client) | `web` |
| Event ingest / normalization | `services/ingest/` (Go) | `ingest-worker` |
| Correlation / risk scoring | `services/fusion/app/` | `fusion` |
| A vendor connector | `services/connectors/` | `connectors` |
| Enrichment providers | `services/enrichment/` (Go) | `enrichment` |

---

## 7. The dev loop

### 7.1 Edit → push → live

```bash
# 1. work on a change in the fork (development branch)
git switch development
# ...edit files...

# 2. ship it
git add -A
git commit -m "feat(api): add /api/v1/foo endpoint"
git push origin development

# 3. Dokploy rebuilds. Two ways:
#    - Auto-deploy webhook enabled → push triggers it automatically, OR
#    - Dokploy UI → service → Deployments → Redeploy
```

Only **changed** services rebuild (BuildKit layer cache), so redeploys are much
faster than the first ~10–15 min cold build. After deploy, verify in the **Logs**
tab.

### 7.2 Confirm your build actually shipped

Because images build from source, a quick sanity check is to log something on
startup or hit your new route. E.g. after adding an endpoint:

```bash
curl -s https://ai-soc.vortiqx.com/api/v1/foo | jq .
```

### 7.3 How the schema stays in sync (important)

On boot, **because `ENVIRONMENT=development`**, `api` self-bootstraps the schema
(`services/api/app/main.py` lifespan):

1. `Base.metadata.create_all` → creates any new ORM tables.
2. `run_migrations.py` → applies every `services/api/migrations/*.sql` **once**,
   tracked in the `aisoc_schema_migrations` table (idempotent, safe to re-run).

**So adding a table = push a migration + redeploy. No manual `migrate` step.**
(The `postgres` container also mounts `migrations/` at initdb, but that only runs
on the *first* boot of an empty volume — the runtime runner is the reliable path.)

---

## 8. Worked examples (matching our goals)

### Example A — Add a backend API endpoint (edit code → appears on Dokploy)

`services/api/app/api/v1/endpoints/foo.py`:

```python
from fastapi import APIRouter
router = APIRouter(prefix="/foo", tags=["foo"])

@router.get("/ping")
async def ping():
    return {"pong": True, "by": "vortiqx"}
```

Register it in `services/api/app/api/v1/router.py`:

```python
from app.api.v1.endpoints import foo
api_router.include_router(foo.router)
```

```bash
git commit -am "feat(api): add /foo/ping" && git push origin development
# redeploy → then:
curl -s https://ai-soc.vortiqx.com/api/v1/foo/ping
# {"pong":true,"by":"vortiqx"}
```

### Example B — Change the agent's LLM model/behavior

The agent resolves its LLM from env via
`services/agents/app/security/llm_resolver.py`. To point a specific run at a
different served model, change `LLM_MODEL` in Dokploy env (e.g. `qwen2.5-14b`)
and redeploy `agents` — no code change. For behavior (prompt, tool logic), edit
the graph under `services/agents/app/` and push; `agents` rebuilds.

### Example C — Inject data directly into Postgres

**Recommended (robust, re-runnable): extend the seeder.** It uses the ORM with
correct tenant/FK wiring and is idempotent. In
`services/api/app/scripts/seed_demo.py`, add rows inside the existing seed flow
(reuse `DEMO_TENANT_ID = 00000000-0000-0000-0000-000000000001`). Push → the
`seed` service re-runs on redeploy.

**Ad-hoc (raw SQL) for quick experiments:**

```bash
psql "postgresql://aisoc:<POSTGRES_PASSWORD>@<VM-IP>:55432/aisoc"

-- inspect the real columns first (schema is wide):
\d alerts

-- then insert against the demo tenant:
INSERT INTO alerts (id, tenant_id, title, severity, status, created_at)
VALUES (gen_random_uuid(),
        '00000000-0000-0000-0000-000000000001',
        'VortiQ-x test alert', 'high', 'open', now());
```

Refresh the console (you're admin, writes on) and the row appears. Always
`\d <table>` first — column sets differ per table and evolve with migrations.

### Example D — Add a new DB table

`services/api/migrations/999_vortiqx_notes.sql`:

```sql
CREATE TABLE IF NOT EXISTS vortiqx_notes (
    id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id  uuid NOT NULL,
    body       text NOT NULL,
    created_at timestamptz NOT NULL DEFAULT now()
);
```

Push → redeploy. On `api` startup the runner applies it once and records it in
`aisoc_schema_migrations`. Write migrations defensively (`IF NOT EXISTS`,
`ADD COLUMN IF NOT EXISTS`) so partial re-applies are safe.

### Example E — Modify the console UI

Pages live under `apps/web/src/app/**`; the API client is
`apps/web/src/lib/api.ts` (calls are same-origin `/api/v1/...`, proxied by
Next.js to `api:8000`). Edit, push → `web` rebuilds. Note the build bakes
`NEXT_PUBLIC_*` values, so anything browser-visible needs a rebuild, not just a
restart.

---

## 9. Operations runbook

| Task | How |
|---|---|
| Redeploy | Dokploy → service → **Deployments → Redeploy** (or push with auto-deploy) |
| Read logs | **Logs** tab, or `docker logs <container> --tail 100 -f` on the VM |
| DB shell | `docker exec -it <postgres-container> psql -U aisoc aisoc` or external `:55432` |
| Re-run the seeder | it re-runs each deploy; or `docker compose run --rm seed` |
| Backup Postgres | `docker exec <postgres> pg_dump -U aisoc aisoc > aisoc-$(date +%F).sql` |
| Wipe & reset | remove the `postgres_data` volume (⚠️ destroys all data) then redeploy |

---

## 10. Security — read before sharing the URL

This is a **dev** deployment: `ENV=development` means **auth bypass is on** and
writes are open. `ai-soc.vortiqx.com` is publicly resolvable through Cloudflare.
Therefore:

- **Put Cloudflare Access (Zero Trust) in front** of `ai-soc.vortiqx.com` (email/SSO),
  or restrict by IP/WAF. Without it, anyone who opens the URL lands as **admin**
  with write access.
- The published Postgres port **55432** is for the **internal dev VM only** —
  never expose it on an internet-facing host.
- For a real, multi-tenant install: `ENV=production`, drop the `55432` publish,
  build `web` with real auth, and use managed secrets.

---

## 11. Troubleshooting (issues we actually hit)

| Symptom | Cause | Fix |
|---|---|---|
| `opensearch` crash-loops | host `vm.max_map_count` too low | `sudo sysctl -w vm.max_map_count=262144` (+ persist in `/etc/sysctl.d/`) |
| `Bind for :::5432 failed: port is already allocated` | another Postgres on the VM owns 5432 | already fixed — host port is **55432** in the compose |
| Domain 404 after adding it | Traefik labels apply only on redeploy | **redeploy** the compose |
| Domain 502 via tunnel | `cloudflared` runs as a container, so its `localhost:80` isn't Traefik | point tunnel Service URL at the host IP, or put `cloudflared` on `dokploy-network` → `http://dokploy-traefik:80` |
| Redirect loop over tunnel | Dokploy domain forcing HTTPS/Let's Encrypt | domain = **HTTP only** (cert: None); let Cloudflare terminate TLS (SSL/TLS mode **Full**) |
| Agent gives generic answers | LLM unreachable → deterministic fallback | check `curl http://192.168.20.12:4000/v1/models` returns 401 from the VM; verify `LLM_API_KEY` |
| Code change didn't appear | pulled cached image / wrong branch | ensure the service has `build:`+`pull_policy: build`, deployed branch is `development`, and you redeployed |

---

## 12. Related internal docs

- vLLM / LiteLLM box (`192.168.20.12:4000`, models `qwen2.5-32b/14b/7b`, `bge-m3`) — the LLM backend this stack points at.
- Dokploy service-DNS conventions.

_Maintainer: VortiQ-x engineering. Keep this in sync with `docker-compose.dokploy.yml`._
