# SMC Bot Backend

FastAPI service that sits between your Android app and your MT5 account
(via MetaApi). It scans markets on a loop, tracks daily P/L against your
SL/TP limits, and exposes exactly the endpoints the Android app already
expects.

## 1. Get your MetaApi credentials

1. Sign up / log in at https://app.metaapi.cloud
2. Add your MT5 account (broker login + password + server) — MetaApi
   gives you back an **account ID**.
3. Generate an **API token** under your MetaApi account settings.

You now have `METAAPI_TOKEN` and `METAAPI_ACCOUNT_ID`.

## 2. Configure

```bash
cp .env.example .env
```

Fill in `.env`:
- `METAAPI_TOKEN`, `METAAPI_ACCOUNT_ID` — from step 1
- `API_KEY` — make up a long random string; this is what the Android app
  sends as `Authorization: Bearer <API_KEY>`
- `SYMBOLS` — must match your broker's exact symbol names (check MT5's
  Market Watch — some brokers suffix symbols, e.g. `XAUUSD.a`)
- `HTF` / `LTF` — timeframes MetaApi understands (`1m`, `5m`, `15m`,
  `1h`, `4h`, `1d`)

## 3. Install and run locally first

```bash
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

Check it's alive: `curl http://localhost:8000/health` → `{"ok": true}`

Check the authenticated endpoints (replace `YOUR_API_KEY`):
```bash
curl -H "Authorization: Bearer YOUR_API_KEY" http://localhost:8000/status
```

**Run this against a demo MT5 account first.** The engine will start
scanning and — once you hit Start from the app (or `POST /bot/start`) —
placing real orders on whatever account MetaApi is connected to.

## 4. What it does

- Every `POLL_INTERVAL_SECONDS`, scans each symbol in `SYMBOLS`:
  structure/swings on the HTF, Turtle Soup + Simple RTO signal checks,
  confluence routing — ported from your `smc_master_bot.py`, with the
  one-shot-per-level and minimum-sweep-distance fixes from your strategy
  tester findings.
- Every triggered, routed signal is recorded (shows up in the app's
  Signals tab) regardless of bot mode.
- Orders are only placed when mode is `RUNNING`.
- Daily P/L is tracked from an equity baseline captured once per UTC
  day. If it breaches your SL or TP limit, the engine closes everything
  and switches to `PAUSED` automatically — same as the app's Close All,
  just triggered by the limit instead of your tap.
- State (mode, daily limits, today's baseline) persists to
  `data/state.json`, so restarting the service doesn't reset anything
  mid-day.

## 5. Endpoints (all except `/health` require the `Authorization` header)

| Method | Path | Purpose |
|---|---|---|
| GET | `/status` | mode, uptime, today's P/L, SL/TP limits |
| GET | `/signals` | recent routed signals |
| GET | `/positions` | live open positions + PnL |
| POST | `/bot/start` | switch to RUNNING |
| POST | `/bot/pause` | switch to PAUSED (keeps scanning, stops trading) |
| POST | `/bot/close-all` | close everything, switch to PAUSED |
| POST | `/settings/daily-limits` | body: `{"stopLoss": 100, "takeProfit": 200}` |

## 6. Hosting — easiest option first

**Railway (no server management at all):**

1. Push this folder to a GitHub repo (or use `railway up` from the CLI to skip GitHub entirely).
2. At [railway.app](https://railway.app): New Project → deploy from that repo.
3. In the project's Variables tab, paste in everything from your `.env`.
4. Railway gives you a live HTTPS URL automatically (e.g.
   `https://your-app.up.railway.app`) — no domain, no cert, no SSH.
5. Use that URL as `baseUrl` in the Android app's `RepositoryProvider.kt`.

Because this app keeps a background scan loop running (not just
request/response), pick Railway's Hobby tier or similar rather than a
free tier that spins down idle services — those will kill the loop
between requests. A couple dollars a month keeps it alive continuously.

**Or a VPS (more control, more setup):**

Any small always-on VPS works (DigitalOcean, Hetzner, etc.).

**Run as a systemd service** so it survives reboots — `/etc/systemd/system/smc-backend.service`:

```ini
[Unit]
Description=SMC Bot Backend
After=network.target

[Service]
WorkingDirectory=/opt/smc-backend
ExecStart=/opt/smc-backend/.venv/bin/uvicorn app.main:app --host 127.0.0.1 --port 8000
Restart=always
EnvironmentFile=/opt/smc-backend/.env

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable smc-backend
sudo systemctl start smc-backend
```

**Put HTTPS in front of it** — the Android app should talk to `https://`,
not plain HTTP, since it's sending trade commands. Easiest option is
Caddy (auto-HTTPS via Let's Encrypt):

```
# /etc/caddy/Caddyfile
your-domain.com {
    reverse_proxy localhost:8000
}
```

## 7. Point the Android app at it

In the Android project, edit
`app/src/main/java/com/smcbot/control/data/RepositoryProvider.kt`:

```kotlin
object RepositoryProvider {
    val instance: BotRepository by lazy {
        RemoteBotRepository(
            baseUrl = "https://your-domain.com/",
            apiKey = "the_same_API_KEY_from_.env"
        )
    }
}
```

## 8. Things worth refining before trusting this with real size

- **Stop-loss/take-profit model** in `engine.py`'s `_execute_signal` is a
  fixed-points placeholder (ported straight from your `phase7_execution.py`
  example). Your MQL5 EA's actual SL/TP logic (structure-based stops,
  OTE-zone entries) is more developed than this — worth porting over
  once the plumbing here is confirmed working.
- **Position sizing** (`_calculate_lot_size`) depends on
  `terminal_state.specification(symbol)` being available from MetaApi;
  double-check this call's exact shape against the MetaApi SDK version
  you actually install, since their SDK has changed method names across
  major versions.
- Only one open position per symbol is assumed when reading back the
  current price in `_execute_signal` — fine for now, but if you add
  pyramiding later this needs revisiting.
- This runs as a single process holding state in memory — don't run
  multiple workers/replicas, they won't share state.
