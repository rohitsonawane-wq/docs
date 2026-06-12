# Deployment Guide — bcd-japan backend

How to run the stack locally, how to deploy it to the dev server, and how to add a
new service. This is the operational companion to `README.md` (what's in the repo)
and `RUNNING.md` (quick run steps).

- [1. Architecture at a glance](#1-architecture-at-a-glance)
- [2. Prerequisites](#2-prerequisites)
- [3. Run locally](#3-run-locally)
- [4. Compose file layout](#4-compose-file-layout)
- [5. Environment & secrets](#5-environment--secrets)
- [6. Deploy to the dev server](#6-deploy-to-the-dev-server)
- [7. Add a new service](#7-add-a-new-service)
- [8. Migrations](#8-migrations)
- [9. Kong routes & JWT](#9-kong-routes--jwt)
- [10. Operations & troubleshooting](#10-operations--troubleshooting)

---

## 1. Architecture at a glance

```
client → nginx (:443 / :3000 / :8444)
            ├─ api-dev.shreetest.in  → Kong (:8000) → service upstreams
            └─ auth-dev.shreetest.in → Keycloak (:8080)

Kong (JWT validation) ──► api-gateway (:3000)   BFF / legacy proxy
                          booking-service  (TCP 3002 / HTTP 4002)
                          lookup-service   (TCP 3003 / HTTP 3013)
                          user-service     (TCP 3001 / HTTP 3011)
                          tenant-policy-service (HTTP 4001)

All app services share ONE Postgres (container bcd_japan_db) with a role-scoped
schema each (see init-schemas.sql). Kong has its own Postgres (kong_database).
Keycloak has its own Postgres. Vault (dev mode) backs per-service secrets.
```

- **Each service is its own git repo**, cloned as a sibling directory inside this
  root and consumed as a Docker `build:` context. This root repo only orchestrates.
- **`bcd-japan-booking-service` is a nested git repo** — branch/commit/PR inside it,
  not the monorepo root.
- **`bcd-japan-kong-api-gateway/` in-tree is stale and not a clone** — edit Kong
  routes via a fresh clone + PR of that repo (the `kong-deck` sidecar builds from it).

---

## 2. Prerequisites

- Docker Desktop / Docker Engine + Compose v2 (`docker compose`, not `docker-compose`)
- Node 22 (only if running services on the host instead of in containers)
- All service repos cloned as siblings inside this root:
  - `bcd-japan-backend-api-gateway/`
  - `bcd-japan-booking-service/`
  - `bcd-japan-lookup-service/`
  - `bcd-japan-user-service/`
  - `bcd-japan-tenant-policy-service/`
  - `bcd-japan-kong-api-gateway/` (for the `kong-deck` sidecar)
- `cp .env.example .env`, then fill in every value (compose reads `${VAR}` with no
  inline defaults — a missing var fails `up`).

---

## 3. Run locally

### Option A — everything in Docker (closest to prod)

```bash
cp .env.example .env
npm run infra:up        # postgres, kong(+db+migrations+deck), keycloak(+db), mailhog, vault(+seed)
docker compose up -d --build api-gateway booking-service lookup-service user-service tenant-policy-service nginx-proxy
```

`infra:up` layers four compose files (`docker-compose.yml` + `keycloak` + `override`
+ `vault`). See the root `package.json` scripts for the exact invocation.

### Option B — infra in Docker, services on the host (fast iteration)

```bash
npm install                 # root (concurrently)
npm run install:all         # installs deps in each service
npm run start:all           # infra:up + start:services (all 5 via concurrently)
```

When services run on the host, Kong must reach them via `host.docker.internal`.
`docker-compose.override.yml` already points the upstream URLs there — confirm
`TPS_UPSTREAM_URL` / `BOOKING_UPSTREAM_URL` / `LOOKUP_UPSTREAM_URL` in `.env` use
`http://host.docker.internal:<port>` for this mode.

### Key local URLs

| URL | What |
|---|---|
| http://localhost:8000 | Kong proxy (send API traffic here) |
| http://localhost:8080 | Keycloak admin console |
| http://localhost:8025 | MailHog UI (dev SMTP) |
| http://localhost:3000 | api-gateway (via nginx) |
| http://localhost:5438 | Postgres (host port → container 5432) |
| http://localhost:8001 | Kong Admin API (localhost-only) |
| http://localhost:8002 | Kong Manager GUI (localhost-only) |

### Stop / reset

```bash
npm run infra:down                  # stop infra (keeps volumes/data)
docker compose down                 # stop app stack
docker compose down -v              # ⚠️ also wipes Postgres/Kong/Keycloak volumes
```

`docker compose down -v` forces `init-schemas.sql` and the Keycloak realm import to
re-run on next boot — use it when DB state or the realm is corrupted, never casually.

---

## 4. Compose file layout

One base file, env-driven overlays — never fork per environment.

| File | Adds |
|---|---|
| `docker-compose.yml` | App services, shared Postgres, Kong (+db, migrations, deck), nginx, Vault |
| `docker-compose.keycloak.yml` | Keycloak + its Postgres + MailHog |
| `docker-compose.override.yml` | Local-dev tweaks (host upstreams, etc.) — auto-loaded by `docker compose` |
| `docker-compose.vault.yml` | Vault dev seeding (`vault-seed`, AppRole IDs) |
| `docker-compose.dev.yml` | Alt dev variant wiring Vault-backed env into services |

`docker compose` auto-merges `docker-compose.override.yml`. Any other overlay must be
passed explicitly with `-f` (as the `npm run infra:*` scripts do).

---

## 5. Environment & secrets

- **Single source of truth:** `.env.example` documents every variable. `.env` is
  git-ignored — never commit it; if a real secret appears in a diff, rotate it.
- **Per-service `.env`:** each service block also `env_file`s its own
  `./bcd-japan-<svc>/.env` (`required: false`). The compose `environment:` block
  **overrides** `env_file` (compose precedence: `environment` > `env_file`), so DB
  host/port are forced to container DNS regardless of what the service `.env` says.
- **Naming gotcha:** lookup-service uses `DB_USERNAME` / `DB_PASSWORD`; every other
  service uses `DB_USER` / `DB_PASS`. Easy to mix up.
- **Vault:** dev mode runs `vault server -dev` (in-memory, root token, auto-unseal) —
  **not production-ready**. For real environments, provision AppRoles externally
  (`scripts/vault-provision.sh`), paste RoleID/SecretID into `.env`, and swap Vault
  for a Raft+TLS overlay. See `docs/hashicorp-vault-*.md`.
- **In non-local environments, change** every `CHANGE_ME` / `*_pass` / `KONG_PG_PASSWORD`
  default and the public `KEYCLOAK_*` / `SWAGGER_*` URLs.

---

## 6. Deploy to the dev server

The dev server is a single EC2 box that clones **this** repo and runs the full stack
in Docker. Public hostnames terminate at nginx:

- `api-dev.shreetest.in` → nginx :443 → Kong → services
- `auth-dev.shreetest.in` → nginx :443 → Keycloak

### Standard deploy (image/code already merged to the deployed branch)

```bash
ssh <dev-server>
cd ~/bcd-japan-backend-root
git pull
# pull the latest for each service repo too (they are separate clones):
for d in bcd-japan-*-service bcd-japan-backend-api-gateway bcd-japan-kong-api-gateway; do
  git -C "$d" pull
done
docker compose -f docker-compose.yml -f docker-compose.keycloak.yml -f docker-compose.vault.yml up -d --build
docker compose ps
```

> There is **no CI/CD pipeline wired yet** — deploys are a manual `git pull` + rebuild
> on the box. Images are built on the server from each service's `build:` context, not
> pulled from a registry. (A future registry-based flow is noted in `README.md`.)

### nginx & TLS

- nginx config is committed at `nginx/default.conf` and mounted read-only into the
  `nginx-proxy` container, with `/etc/letsencrypt` mounted for certs.
- **Important:** the 443 vhosts live in the committed `nginx/default.conf`. Do **not**
  hand-edit nginx on the server only — a later `git pull`/recreate reverts it and
  breaks public HTTPS. Edit the committed file and redeploy. See
  `docs/dev-server-incident-2026-06-10.md`.
- Certs (`auth-dev` / `api-dev` / `bcd-japan-dev`) are Let's Encrypt under
  `/etc/letsencrypt/live/...`. Renew with the host's certbot; nginx mounts them read-only.

### Kong Admin / Manager access

Admin (`:8001`) and Manager (`:8002`) are bound to `127.0.0.1` only — never public.
Reach them over an SSH tunnel:

```bash
ssh -L 8001:127.0.0.1:8001 -L 8002:127.0.0.1:8002 <dev-server>
# then browse http://localhost:8002 locally
```

### After deploy — sync Kong routes/JWT

If service upstreams or issuers changed, re-run the deck sidecar:

```bash
npm run kong:sync       # docker compose run --rm kong-deck
```

---

## 7. Add a new service

Example: `bcd-japan-foo-service`, HTTP port `4005`, DB schema `foo`. Swap names/ports
for the real service. There are **6 files to touch** — miss one and you get the
predictable failure noted under each step. Steps 7–8 (Keycloak/nginx) are optional.

### 7.1 The service repo + Dockerfile

Build the service repo following the DDD/Clean-Arch conventions in `CLAUDE.md` (TPS
`client-management` is the reference; `/scaffold` can generate the skeleton). Clone it
as a sibling dir here. It needs a multi-stage `Dockerfile` matching the others:

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci
COPY tsconfig.json tsconfig.build.json nest-cli.json ./
COPY src/ src/
RUN npm run build

FROM node:22-alpine
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci --omit=dev
COPY --from=build /app/dist ./dist
EXPOSE 4005
CMD ["node", "dist/main"]
```

### 7.2 `init-schemas.sql` — Postgres role + schema

All app services share one Postgres; each gets a role-scoped schema. Append an
idempotent block:

```sql
CREATE SCHEMA IF NOT EXISTS foo;
DO $$
BEGIN
  IF NOT EXISTS (SELECT FROM pg_roles WHERE rolname = 'foo_svc') THEN
    CREATE ROLE foo_svc WITH LOGIN PASSWORD 'foo_pass';
  END IF;
END $$;
GRANT USAGE, CREATE ON SCHEMA foo TO foo_svc;
ALTER DEFAULT PRIVILEGES IN SCHEMA foo GRANT ALL ON TABLES TO foo_svc;
ALTER DEFAULT PRIVILEGES IN SCHEMA foo GRANT ALL ON SEQUENCES TO foo_svc;
```

> ⚠️ `init-schemas.sql` runs **only on a fresh Postgres volume** (it's a
> `docker-entrypoint-initdb.d` script). On an existing DB, `psql` the block in
> manually or `docker compose down -v` (wipes all data). **Skip this → the service
> crashes on boot with a "role/schema does not exist" auth error.**

### 7.3 `.env` + `.env.example` — add the vars

```dotenv
FOO_DB_USER=foo_svc
FOO_DB_PASS=foo_pass
FOO_DB_SCHEMA=foo
FOO_SERVICE_PORT=4005
FOO_UPSTREAM_URL=http://foo_service:4005   # how Kong reaches it (container DNS)
```

> Compose reads `${VAR}` with **no inline defaults** — a missing var fails `up`
> immediately. Add to both files (`.env` to run, `.env.example` to document).

### 7.4 `docker-compose.yml` — the service block

Copy the `booking-service` block and rename. `environment:` **overrides** `env_file:`,
which is why DB host/port are forced here to container DNS regardless of the service's
own `.env`:

```yaml
  foo-service:
    build: ./bcd-japan-foo-service
    container_name: foo_service
    restart: unless-stopped
    env_file:
      - path: ./bcd-japan-foo-service/.env
        required: false
    environment:
      NODE_ENV: ${NODE_ENV}
      DB_HOST: ${DB_HOST}          # "postgres" — the container, not localhost
      DB_PORT: ${DB_PORT}          # 5432 (container-internal, not the 5438 host port)
      DB_NAME: ${POSTGRES_DB}
      DB_USER: ${FOO_DB_USER}
      DB_PASS: ${FOO_DB_PASS}
      DB_SCHEMA: ${FOO_DB_SCHEMA}
      PORT: ${FOO_SERVICE_PORT}
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - services_net
```

Get these right:
- **No `ports:` block** unless you genuinely need host access. Services are reached on
  `services_net` and routed through Kong — publishing a port bypasses Kong (and its
  JWT check).
- **`networks: [services_net]` is mandatory** — omit it and Kong/Postgres can't resolve
  `foo_service` by DNS.
- Copy from `booking`/`user`, **not `lookup`** — lookup-service uses
  `DB_USERNAME`/`DB_PASSWORD` while everyone else uses `DB_USER`/`DB_PASS`.

### 7.5 Root `package.json` — orchestrator scripts

```jsonc
"start:foo": "npm --prefix bcd-japan-foo-service run start:dev",
```

Then append `foo-service` to:
- `install:all` → `… && npm --prefix bcd-japan-foo-service install`
- `start:services` → add `"npm:start:foo"` plus a name/color to the `concurrently` flags
- `infra:up` service list **only if** it should boot with infra (usually no — app
  services aren't infra)

### 7.6 Kong route + JWT

Kong is what makes the service callable at `api-dev.shreetest.in`. In the
**`bcd-japan-kong-api-gateway`** repo (a separate clone — the in-tree dir is stale):
- add the route + upstream for `foo-service` to the decK config, and
- wire `FOO_UPSTREAM_URL` into the `kong-deck` block's `environment:` in
  `docker-compose.yml`, mirroring `TPS_UPSTREAM_URL`/`BOOKING_UPSTREAM_URL`.

Then `npm run kong:sync` (see §9). For host-run services (Option B), set
`FOO_UPSTREAM_URL=http://host.docker.internal:4005` instead of the container DNS.

### 7.7 Keycloak (optional)

Only if the service needs its own client/roles — add them to
`keycloak/realm-export.json` (use the `/keycloak-realm` skill) and apply with
`scripts/keycloak-live-apply.sh`.

### 7.8 nginx (optional)

Only if the service needs a dedicated public hostname; otherwise it's reached through
Kong via `api-dev.shreetest.in`.

### Bring it up

```bash
docker compose up -d --build foo-service
docker compose logs -f foo-service
npm run kong:sync
```

---

## 8. Migrations

- **TypeORM across all services, `synchronize: false`.** Schema changes ship as SQL
  under each service's `migrations/`, applied per service.
- `booking-service` has a `booking-migrate` one-shot sidecar that applies its SQL on
  boot and exits; `booking-service` won't start if it fails. Re-run after adding SQL:
  ```bash
  docker compose up -d --build booking-migrate booking-service
  ```
- Migrations must be **idempotent** (`IF NOT EXISTS` / `CREATE OR REPLACE`) — verify by
  running the file twice on a scratch Postgres. The `/migrate` skill authors/verifies.
- "relation already exists" on `booking-migrate` means the SQL isn't idempotent — fix
  the guards, or `docker compose down -v` to start clean.

---

## 9. Kong routes & JWT

- Kong validates Keycloak-issued JWTs at the gateway; services trust the gateway and
  derive `tenantId` / `createdBy` from the token.
- The `kong-deck` sidecar (built from `bcd-japan-kong-api-gateway`) fetches the realm
  JWKS and syncs routes + one `jwt_secret` per issuer in `ISSUERS`.
- **All issuers in `ISSUERS` must be minted by the SAME Keycloak** (they share one
  fetched signing key). Don't mix issuers from different Keycloak instances.
- `REFRESH_INTERVAL` (seconds) controls the refresh loop; empty = one-shot sync on `up`.
  A stale upstream / "no Route matched" is usually fixed by recreating `kong-deck`.
- Manual sync: `npm run kong:sync`.

---

## 10. Operations & troubleshooting

```bash
docker compose ps                          # service status
docker compose logs -f <service>           # tail one service
curl http://localhost:3000/health          # gateway aggregate health
curl http://localhost:3012/health/live     # booking liveness
```

| Symptom | Fix |
|---|---|
| `up` fails on a missing `${VAR}` | A var isn't in `.env`. Diff against `.env.example`. |
| Kong "no Route matched" / wrong upstream | Re-run `npm run kong:sync`; check `*_UPSTREAM_URL` (container DNS in Docker, `host.docker.internal` for host-run services). |
| api-gateway can't reach Keycloak | Confirm Keycloak up (`curl localhost:8080/health/ready`); check `host.docker.internal` resolves in-container. |
| 401 through Kong | Use `/auth-doctor` to mint/decode a dev token and find which layer rejects it. Verify the token's `iss` is in `ISSUERS`. |
| Port 5438 in use | Change `POSTGRES_HOST_PORT` in `.env` (host-side mapping only). |
| Public HTTPS down after a pull | nginx 443 vhosts must be in committed `nginx/default.conf`; recreate `nginx-proxy`. See `docs/dev-server-incident-2026-06-10.md`. |
| `booking-migrate` "relation already exists" | SQL not idempotent — add guards or `down -v`. |

### Related docs

- `README.md` — repo contents & service map
- `RUNNING.md` — minimal run steps
- `ARCHITECTURE.md` — system design
- `docs/hashicorp-vault-*.md` — Vault setup
- `docs/kong-keycloak-jwks-refresh-plan.md` — deck sidecar / JWT refresh
- `docs/dev-server-incident-2026-06-10.md` — nginx 443 incident postmortem
