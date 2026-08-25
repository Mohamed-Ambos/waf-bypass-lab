# WAF Bypass Lab

A self-contained Docker lab for practising web bug hunting **against a Web Application
Firewall** — the way real targets are actually protected. A deliberately-vulnerable app sits
behind **two different WAFs** *and* is exposed directly, so you can prove a payload works, then
learn to slip it past each WAF and see exactly *why* it was blocked.

- **OWASP CRS / ModSecurity** — the classic regex + anomaly-score engine (no GUI; you read its
  audit log).
- **Chaitin SafeLine** — a semantic-analysis WAF with a full **admin dashboard** (live traffic,
  attack-event log, point-and-click block/allow rules).
- **Direct door** — the raw app, no WAF, as a control to confirm a payload is valid first.

> ⚠️ The target is **intentionally vulnerable**. Everything binds to `127.0.0.1` only —
> **never expose it to a network you don't fully control.** For authorised learning / CTF use.

---

## Architecture — the "multiple doors"

The same Juice Shop instance is reachable through several front doors. You attack a WAF door,
compare it against the direct door, and read the WAF's verdict to craft a bypass.

```
                         ┌──────────────────────── Docker host · 127.0.0.1 ────────────────────────┐
                         │                                                                          │
    ATTACKER             │   :8080    ┌───────────────────────────┐                                 │
    ───────────────────► │ ─────────► │  nginx + ModSecurity v3   │ ──┐                             │
    payloads / tools     │            │      + OWASP CRS          │   │                             │
                         │            └───────────────────────────┘   │                             │
                         │   :9080    ┌───────────────────────────┐   │      ┌───────────────────┐  │
    ───────────────────► │ ─────────► │  SafeLine  (tengine +     │ ──┼────► │  OWASP Juice Shop  │  │
                         │            │   semantic detector)      │   │      │   (vulnerable)     │  │
                         │            └───────────────────────────┘   │      └───────────────────┘  │
                         │   :3000                                     │             ▲ :3000         │
    ───────────────────► │ ────────────────────────────────────────────┘   direct (control/oracle) │
    control / oracle     │                                                                          │
                         │            ─────────── control plane (not in traffic path) ───────────   │
    ADMIN (browser)      │   :9443    ┌───────────────────────────┐                                 │
    ───────────────────► │ ─────────► │  SafeLine dashboard (mgt)  │ ◄──► Postgres (rules, events)   │
    configure / observe  │  (HTTPS)   └───────────────────────────┘                                 │
                         └──────────────────────────────────────────────────────────────────────────┘
```

| Door | URL | Engine | Config / visibility |
|------|-----|--------|---------------------|
| **Direct** (control) | http://localhost:3000 | none | — the raw app |
| **CRS WAF** | http://localhost:8080 | ModSecurity v3 + OWASP CRS (regex + anomaly score) | env vars · `logs/audit.log` |
| **SafeLine WAF** | http://localhost:9080 *(you create this port in the dashboard)* | semantic analysis | **dashboard** at `https://127.0.0.1:9443` |

The CRS WAF ships in the main `docker-compose.yml`. SafeLine is an optional second stack in
[`safeline/`](safeline/) — see [its README](safeline/README.md) for the full walkthrough.

---

## Requirements

- Docker + Docker Compose v2 (`docker compose version`)
- CRS lab: ~1 GB RAM, ~1.3 GB disk. SafeLine adds ~1 GB of images and 7 containers.

---

## Quick start — the CRS lab

```bash
docker compose up -d        # pull + start the WAF and the app
docker compose ps           # wait until both report "healthy"
docker compose logs -f waf  # watch the WAF
docker compose stop         # pause (keeps containers)   ·  down = stop & remove
```

Open in a browser:

- **Through the WAF:** http://localhost:8080  ← attack this
- **Direct (no WAF):** http://localhost:3000  ← control · scoreboard at `/#/score-board`

> **First run — log permissions.** The WAF writes its audit log as uid `101`, but a fresh
> `git clone` creates `logs/` owned by your user, so the file stays empty. Fix once with
> `chmod 777 logs` (the `logs/` contents are gitignored anyway), then
> `docker compose restart waf`.

---

## The workflow

The whole point of the lab, in four steps:

1. **Prove the bug on `:3000`.** Land a working exploit against the app directly. Now you
   *know* the payload is valid — the app isn't patched.
2. **Replay it on a WAF door** (`:8080` CRS, or `:9080` SafeLine). A block returns `403`. The
   difference between `:3000` (200) and the WAF door (403) is the WAF, not the app.
3. **Read *why* it blocked** — CRS gives you a rule ID + anomaly score in `logs/audit.log`;
   SafeLine gives you a semantic verdict on its dashboard's Events page.
4. **Craft a bypass** (encoding, case, comments, JSON vs form body, chunking, header tricks)
   until the WAF door lets it through too — then compare: a trick that beats CRS often fails
   against SafeLine's semantic engine, and vice-versa.

### Quick sanity check that a WAF is really blocking

```bash
# WAF door -> expect 403
curl -s -o /dev/null -w '%{http_code}\n' "http://localhost:8080/?x=<script>alert(1)</script>"
# Direct door -> expect 200
curl -s -o /dev/null -w '%{http_code}\n' "http://localhost:3000/?x=<script>alert(1)</script>"
```

---

## CRS WAF — reading and tuning

### See which rules fired (live)

```bash
tail -f logs/audit.log | jq -r '.transaction.messages[]? | "\(.details.ruleId)  \(.message)"'
```

Example output after firing an XSS and a SQLi payload:

```
941100  XSS Attack Detected via libinjection
941110  XSS Filter - Category 1: Script Tag Vector
942100  SQL Injection Attack Detected via libinjection
949110  Inbound Anomaly Score Exceeded (Total Score: 15)
```

CRS rule-ID families you'll see a lot: `941xxx` XSS · `942xxx` SQLi · `930xxx` LFI/path
traversal · `932xxx` RCE/command injection.

### Tuning knobs (edit `docker-compose.yml`, then `docker compose up -d`)

| Env var | Effect |
|---|---|
| `MODSEC_RULE_ENGINE` | `On` blocks · `DetectionOnly` logs-but-allows (study rules without being stopped) · `Off` |
| `PARANOIA` | `1`–`4`. Higher = stricter rules = more to bypass, and more false positives. Start at `1`. |
| `ANOMALY_INBOUND` | Blocking threshold. CRS scores each matched rule; the request blocks when the sum reaches this. Lower = twitchier. |

Tip: set `MODSEC_RULE_ENGINE=DetectionOnly` to see *everything a payload trips* without a 403
stopping your tooling — then switch back to `On` to practise beating it.

---

## SafeLine WAF — the dashboard door

SafeLine is the WAF with a real admin console — the "how do admins configure this?" experience.
It's a separate stack so the CRS lab stays untouched.

```bash
cd safeline
docker compose --env-file .env up -d                       # first run pulls ~1 GB of images
docker exec safeline-mgt /app/mgt-cli reset-admin --once   # prints a one-time admin login
```

Then open **https://127.0.0.1:9443** (accept the self-signed cert), log in, and add Juice Shop
as a **protected site**: domain `127.0.0.1`, listen port `9080`, upstream `http://127.0.0.1:3000`.
Traffic to `http://localhost:9080` now flows through SafeLine and appears live in the dashboard.

Stop it when you're done (data in `safeline/data/` survives):

```bash
docker compose --env-file .env stop
```

**Why run both:** CRS matches *regex signatures* and sums an anomaly score; SafeLine *parses*
the payload semantically. Signature-evasion tricks (case, comments, encoding) that slip past CRS
often don't fool SafeLine — and understanding that gap is the real lesson. Full details, the
dashboard tour, and an end-to-end test are in [`safeline/README.md`](safeline/README.md).

---

## Repository layout

```
.
├── docker-compose.yml     # CRS lab: waf (ModSecurity/CRS) + juiceshop
├── logs/                  # ModSecurity JSON audit log (runtime, gitignored)
├── safeline/              # optional SafeLine WAF stack (dashboard door)
│   ├── compose.yaml       # vendored from waf.chaitin.com; dashboard bound to 127.0.0.1
│   ├── .env               # local config incl. generated password (gitignored)
│   ├── data/              # SafeLine runtime data (gitignored)
│   └── README.md          # SafeLine setup, dashboard tour, three-door loop
├── CLAUDE.md              # guidance for AI coding assistants
└── README.md
```

## Suggested tools

Burp Suite (proxy/repeater/intruder), `sqlmap`, `ffuf`, `nikto` — always with `:3000` as the
control target to confirm findings before you blame the WAF.

## Scope

- **App:** OWASP Juice Shop — 100+ challenges with a built-in scoreboard (`/#/score-board`).
- **WAFs:** OWASP CRS on ModSecurity v3 / nginx, and Chaitin SafeLine (semantic engine).

Everything binds to `127.0.0.1` only. This app is intentionally vulnerable — **never expose it
to a network you don't fully control.** For authorised learning / CTF use only.
