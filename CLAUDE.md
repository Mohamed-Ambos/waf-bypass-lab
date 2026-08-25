# CLAUDE.md

Guidance for Claude Code (claude.ai/code) when working in this repository.

## What this project is

**WAF Bypass Lab** — a small, self-contained Docker Compose lab for practising web
application bug hunting **against a Web Application Firewall**. It is a personal
learning/CTF-style project and is **not related to any other codebase**; treat it in
isolation.

Two containers, defined in `docker-compose.yml`:

| Service | Image | Role |
|---------|-------|------|
| `waf` | `owasp/modsecurity-crs:nginx` | nginx + ModSecurity v3 + OWASP Core Rule Set, reverse-proxying the app. Exposed on `127.0.0.1:8080`. |
| `juiceshop` | `bkimminich/juice-shop:latest` | Intentionally-vulnerable target app. Exposed **directly** on `127.0.0.1:3000` (bypasses the WAF) as a control. |

The design idea is **"multiple doors"** into the *same* Juice Shop: the app is reachable
through a WAF and directly (`:3000`). You confirm a payload works on `:3000` (the control),
then learn to get it past a WAF door and read *why* it blocked. See `README.md` for the full
workflow and architecture diagram.

**Second WAF — SafeLine (`safeline/`), the dashboard door.** A GUI-driven WAF (Chaitin
SafeLine, semantic engine) vendored as its own compose stack in `safeline/`, with an admin
console on `https://127.0.0.1:9443`. It is a **separate compose project** from the main lab and
reaches Juice Shop at `http://127.0.0.1:3000` because its `tengine` proxy runs
`network_mode: host` (so the main `docker-compose.yml` is untouched). Key facts:

- **The `:9080` attack door does not exist until you create it** in the dashboard as a protected
  site: domain `127.0.0.1`, listen `9080`, upstream `http://127.0.0.1:3000`.
- **`safeline/compose.yaml` is vendored verbatim from `waf.chaitin.com`** with a *single*
  deliberate change: the dashboard port is bound to `127.0.0.1` (loopback rule). If you re-pull
  upstream, re-apply that one edit.
- **Subnet:** `SUBNET_PREFIX=172.30.222` in `safeline/.env`. The upstream default `172.22.222`
  collides with an existing `infisical` docker network on this host — keep it off `172.22.x`.
- **Engine contrast (the pedagogical point):** SafeLine does semantic analysis, so there is **no
  rule ID / anomaly score** like CRS. Signature-evasion tricks (case, comments, encoding) that
  beat CRS often fail here, and vice-versa.
- **`safeline/data/` and `safeline/.env` are gitignored** (`.env` holds a generated Postgres
  password). `compose.yaml` and `safeline/README.md` are committed.
- **Heads-up:** SafeLine's containers use `restart: always`, so they auto-start on Docker
  daemon restart / reboot. Switch to `unless-stopped` if that's unwanted.

## Common commands

```bash
docker compose up -d            # start both containers
docker compose ps               # both should report "healthy"
docker compose logs -f waf      # WAF logs
docker compose down             # stop & remove containers (images cached)

# is the WAF actually blocking? (expect 403 vs 200)
curl -s -o /dev/null -w '%{http_code}\n' "http://localhost:8080/?x=<script>alert(1)</script>"
curl -s -o /dev/null -w '%{http_code}\n' "http://localhost:3000/?x=<script>alert(1)</script>"

# which CRS rules fired (JSON audit log, bind-mounted to ./logs)
tail -f logs/audit.log | jq -r '.transaction.messages[]? | "\(.details.ruleId)  \(.message)"'
```

SafeLine stack (run from the `safeline/` dir; `--env-file .env` is required every time):

```bash
cd safeline
docker compose --env-file .env up -d                       # start (first run pulls ~1 GB)
docker compose --env-file .env ps                          # 7 containers; several report healthy
docker exec safeline-mgt /app/mgt-cli reset-admin --once   # mint a one-time admin login
curl -sk -o /dev/null -w '%{http_code}\n' https://127.0.0.1:9443/api/open/health  # expect 200
docker compose --env-file .env stop                        # pause (data in ./data survives)
```

## Validating changes

This is an infra/config repo — there is **no build, lint, or test suite**. "Validation" means:

1. **Lint the compose file:** `docker compose config` (catches YAML / env-var typos before a
   recreate).
2. **Recreate and check health:** `docker compose up -d && docker compose ps` — both services
   should report `healthy`.
3. **The core test loop** is the `403` (WAF) vs `200` (direct) diff: after any change to WAF
   behaviour, re-run the two `curl` sanity checks above and compare the status codes. A change
   that makes `:8080` stop returning `403` on a known-malicious payload has weakened the WAF; a
   change that makes `:3000` return anything but `200` has broken the app.

**Which reload do I need?**

- **Changed an env var / port / volume in `docker-compose.yml`** → `docker compose up -d`
  (recreates the affected container so the new config takes effect).
- **Changed `logs/` permissions or want a stale audit handle reopened** → `docker compose
  restart waf` (the WAF opens the audit-log file handle at start, so a plain perms fix alone
  won't take until it restarts).

## Conventions & constraints

- **Ports bind to `127.0.0.1` only.** The target is deliberately vulnerable; never expose it
  to an untrusted network. Keep any new service loopback-bound.
- **`logs/` is runtime-only** and gitignored (except `logs/.gitkeep`). Don't commit audit logs.
- **Log permissions gotcha:** the WAF runs as uid `101`; a freshly-cloned `logs/` is owned by
  your user, so the audit file stays empty until you `chmod 777 logs` once. If the file exists
  but is stale, the WAF opens the handle at start — `docker compose restart waf` after changing
  perms.
- **Juice Shop is a distroless image** (only `/nodejs/bin/node`, no shell/wget/curl). The
  healthcheck therefore calls `/nodejs/bin/node` by absolute path — keep that if you edit it.
- **WAF behaviour is configured entirely via environment variables** in `docker-compose.yml`
  (`MODSEC_RULE_ENGINE`, `PARANOIA`, `ANOMALY_INBOUND`, …) — prefer changing those over baking
  custom rule files, unless the goal is specifically to practise writing/whitelisting rules.
- **Pin nothing to `latest` in spirit only:** images use rolling tags for convenience; if a pull
  breaks the lab, pin to a known-good digest rather than debugging upstream.
- This is a learning project for the owner — when explaining WAF/CRS behaviour, show *why* a
  payload was blocked (rule ID, anomaly score, paranoia level), not just that it was.

## Extending the lab

- ✅ **Done:** a second WAF (SafeLine) to compare regex/CRS vs semantic detection — see `safeline/`.
- Add a second vulnerable target (e.g. DVWA) as another compose service behind the same WAF.
- Add a second WAF port running `MODSEC_RULE_ENGINE=DetectionOnly` to diff block-vs-detect.
- Point SafeLine at a *second* protected site so one dashboard fronts multiple targets.

Keep additions self-contained in this repo; do not add dependencies on external/private infra.
