# WAF Bypass Lab

A self-contained Docker lab for practising web bug hunting **against a WAF** — the way
real targets are usually protected. An OWASP CRS / ModSecurity WAF sits in front of a
deliberately-vulnerable app, and the same app is *also* exposed directly so you can prove
a payload works before you fight the WAF.

```
        :8080  ┌─────────────────────────┐         ┌──────────────────┐
  you ───────► │ nginx + ModSecurity v3  │ ──────► │  OWASP Juice Shop │
   (attack)    │      + OWASP CRS  (WAF)  │         │   (vulnerable)    │
               └─────────────────────────┘         └──────────────────┘
                                                             ▲
  you ───────────────────────────────────────────────────────┘
        :3000   DIRECT door — bypasses the WAF (control / oracle)
```

## Requirements

- Docker + Docker Compose v2 (`docker compose version`)
- ~1 GB free RAM, ~1.3 GB disk for images

## Start / stop

```bash
docker compose up -d        # pull + start both containers
docker compose ps           # wait until both are "healthy"
docker compose logs -f waf  # watch the WAF
docker compose down         # stop & remove (images stay cached)
```

> **First run — log permissions.** The WAF writes its audit log as uid `101`, but a fresh
> `git clone` creates `logs/` owned by your user, so the WAF can't write into it and the file
> stays empty. Fix once with `chmod 777 logs` (the `logs/` contents are gitignored anyway).

Open the app in a browser:

- **Through the WAF:** http://localhost:8080  ← attack this
- **Direct (no WAF):** http://localhost:3000  ← control

## The "two doors" workflow

This is the whole point of the lab:

1. **Prove the bug on `:3000`.** Land a working exploit against the app directly. Now you
   *know* the payload is valid — the app isn't patched.
2. **Replay it on `:8080`.** If the WAF blocks it you get a `403`. The difference between
   `:3000` (200) and `:8080` (403) is the WAF, not the app.
3. **Read *why* it blocked.** Every block is written to `logs/audit.log` with the CRS
   **rule ID** and **anomaly score**.
4. **Craft a bypass** (encoding, case, comments, JSON vs form body, chunking, header tricks)
   until `:8080` lets it through too.

### Quick sanity check that the WAF is really blocking

```bash
# WAF door -> expect 403
curl -s -o /dev/null -w '%{http_code}\n' "http://localhost:8080/?x=<script>alert(1)</script>"
# Direct door -> expect 200
curl -s -o /dev/null -w '%{http_code}\n' "http://localhost:3000/?x=<script>alert(1)</script>"
```

### See which rules fired (live)

```bash
# rule id + human message for every rule that fired
tail -f logs/audit.log | jq -r '.transaction.messages[]? | "\(.details.ruleId)  \(.message)"'
```

Example output after firing an XSS and a SQLi payload:

```
941100  XSS Attack Detected via libinjection
941110  XSS Filter - Category 1: Script Tag Vector
942100  SQL Injection Attack Detected via libinjection
949110  Inbound Anomaly Score Exceeded (Total Score: 15)
```

CRS rule-ID families you'll see a lot: `941xxx` XSS, `942xxx` SQLi, `930xxx` LFI/path
traversal, `932xxx` RCE/command injection.

## Tuning knobs (edit `docker-compose.yml`, then `docker compose up -d`)

| Env var | Effect |
|---|---|
| `MODSEC_RULE_ENGINE` | `On` blocks · `DetectionOnly` logs-but-allows (study rules without being stopped) · `Off` |
| `PARANOIA` | `1`–`4`. Higher = stricter rules = more to bypass, and more false positives. Start at `1`. |
| `ANOMALY_INBOUND` | Blocking threshold. CRS scores each matched rule; the request blocks when the sum reaches this. Lower = twitchier. |

Tip: set `MODSEC_RULE_ENGINE=DetectionOnly` to see *everything a payload trips* without a
403 stopping your tooling — then switch back to `On` to practise beating it.

## Suggested tools to point at `:8080`

Burp Suite (proxy/repeater/intruder), `sqlmap`, `ffuf`, `nikto` — with `:3000` as the
control target to confirm findings.

## Targets & scope

- App: **OWASP Juice Shop** — 100+ challenges with a built-in scoreboard (`/#/score-board`).
- WAF: **OWASP CRS** on **ModSecurity v3 / nginx**.

Everything binds to `127.0.0.1` only. This app is intentionally vulnerable — **never expose
it to a network you don't fully control.** For authorised learning/CTF use only.
