# SafeLine WAF — admin-dashboard door

This is a second WAF for the lab, added **because it has a real admin console** (traffic
graphs, an attack-event log, and point-and-click block/allow rules) — the thing you asked to
see. It sits *alongside* the ModSecurity/CRS WAF, not replacing it, so the lab now has three
doors into the same Juice Shop:

| Door | URL | What it is |
|------|-----|-----------|
| Direct | http://127.0.0.1:3000 | app, no WAF (control / oracle) |
| CRS WAF | http://127.0.0.1:8080 | nginx + ModSecurity + OWASP CRS (regex/rule engine, no GUI) |
| **SafeLine WAF** | http://127.0.0.1:9080 *(you create this port in the GUI)* | semantic-detection WAF **with dashboard** |
| SafeLine dashboard | https://127.0.0.1:9443 | the admin console itself |

> **Engine difference worth knowing:** CRS matches **regex signatures** and sums an *anomaly
> score* (you see rule IDs like `941100`). SafeLine uses a **semantic analysis** engine — it
> parses the payload the way the app would and decides if it's an attack, so classic
> signature-evasion tricks (case, comments, encoding) behave very differently against it. That
> contrast is the whole reason to run both.

## How it's wired

- Vendored from SafeLine's official `compose.yaml` (fetched from `waf.chaitin.com`). Only change
  from upstream: the dashboard port is bound to `127.0.0.1` (lab convention — never expose the
  vulnerable stack).
- Config lives in `.env` (gitignored — it holds a generated Postgres password). Runtime data
  lives in `safeline/data/` (gitignored).
- The `safeline-tengine` proxy runs with `network_mode: host`, so its upstream target
  `http://127.0.0.1:3000` reaches Juice Shop's direct door with **no change to the main
  `docker-compose.yml`**.

## Start / stop

```bash
cd safeline
docker compose --env-file .env up -d     # first run pulls ~1 GB of images
docker compose --env-file .env ps        # wait for containers to be healthy
docker compose --env-file .env down      # stop (data in ./data survives)
```

## First login

SafeLine has no default password — you mint a one-time admin credential from the CLI:

```bash
docker exec safeline-mgt /app/mgt-cli reset-admin --once
```

This prints a **username (`admin`)** and a **one-time password**. Then:

1. Open **https://127.0.0.1:9443** (accept the self-signed-cert warning).
2. Log in with those credentials; set a real password when prompted.

## Put Juice Shop behind SafeLine (the admin experience)

In the dashboard, add a **protected site** (menu is usually *Sites* / *防护应用*):

| Field | Value |
|-------|-------|
| Domain / server name | `127.0.0.1` |
| Listen port | `9080` (any free port; this becomes the SafeLine door) |
| Upstream / backend | `http://127.0.0.1:3000` |
| SSL | off |

Save. Now traffic to **http://127.0.0.1:9080** flows *through* SafeLine to Juice Shop, and every
request shows up in the dashboard.

### Prove it end-to-end

```bash
# direct door (no WAF)  -> 200
curl -s -o /dev/null -w 'DIRECT   %{http_code}\n' -H 'Host: 127.0.0.1' "http://127.0.0.1:3000/?x=<script>alert(1)</script>"
# SafeLine door         -> 403 (and an event appears in the dashboard)
curl -s -o /dev/null -w 'SAFELINE %{http_code}\n' -H 'Host: 127.0.0.1' "http://127.0.0.1:9080/?x=<script>alert(1)</script>"
```

## What to explore in the console (this is "how admins configure it")

- **Dashboard / 数据大盘** — live QPS, request/attack counts, top attacked sites, world map.
- **Events / 攻击事件** — every blocked request with the decoded payload and *why* it was flagged.
  Click one after firing a payload to see SafeLine's reasoning (contrast with CRS rule IDs).
- **Protection rules / 防护规则** — toggle the semantic engine, set the action (observe vs block).
- **Access control / allow-lists** — hand-write allow/deny rules, IP allow/block lists, and
  **rate-limiting** / CC protection. This is the "block this, allow that" workflow you wanted.
- **Auth challenge / 人机验证** — make a path require a challenge before it reaches the app.

## The three-door bug-hunting loop

1. Land a payload on **:3000** (prove the app is vulnerable).
2. Replay on **:8080** (CRS) — read `logs/audit.log` for the rule ID + anomaly score.
3. Replay on **:9080** (SafeLine) — read the **Events** page for the semantic verdict.
4. Craft a bypass and compare: a trick that beats CRS often fails against SafeLine's semantic
   engine, and vice-versa. Understanding *why* is the point.
