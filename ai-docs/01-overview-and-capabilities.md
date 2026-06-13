# OpenAlgo — Overview & Capabilities

Platform version **`2.0.1.3`** · Flask 3.1 backend + React 19 frontend · Python 3.12+ · self-hosted, single-user.

---

## 1. What it is, in one paragraph

OpenAlgo turns 30+ Indian brokers (and Delta Exchange crypto) into **one unified REST/WebSocket API** and wraps it with four user-facing products: a broker-agnostic trading API, an in-browser Python strategy host, a no-code visual strategy builder, and a 12-tool options analytics suite. It runs as a **single self-hosted process** (Gunicorn + eventlet, one worker) that holds exactly one broker session, streams one market-data feed, and serves a modern SPA. There is no SaaS component and no multi-user model — server access equals full control, which is the security model.

---

## 2. The runtime, concretely

- **Process model:** production runs `gunicorn --worker-class eventlet -w 1`. Single worker is mandatory (SocketIO/WebSocket state is in-process). Eventlet monkey-patches the stdlib → **no `asyncio`** in request paths; async work uses green threads or a dedicated OS thread.
- **Dev model:** `uv run app.py` uses the Flask dev server with standard threading (no monkey-patching). Code must work in both.
- **Ports:** Flask `5000`, market-data WebSocket proxy `8765`, internal ZeroMQ bus `5555`.
- **Daily lifecycle:** Indian broker tokens expire ~03:00 IST; session management is aligned to this. From **April 1, 2026** SEBI mandates broker-side static-IP whitelisting for transactional orders — so the OpenAlgo server's IP becomes the trust anchor.
- **Package management:** `uv` only (never global Python). Two independent versions live in the repo: the **platform version** (`utils/version.py`) and the **OpenAlgo Python SDK pin** (`openalgo==…` on PyPI).

---

## 3. What the platform can do — capability catalogue

### 3.1 Unified Broker API (`/api/v1/`, 49 endpoint modules)

A single, broker-agnostic schema in front of all brokers, auto-documented via Swagger at `/api/docs`. Auth by API key (in JSON body or `X-API-KEY` header), hashed with a pepper (Argon2) before storage, per-endpoint rate-limited.

- **Order management:** `placeorder`, `placesmartorder` (position-aware), `basketorder`, `splitorder` (iceberg), `modifyorder`, `cancelorder`, `cancelallorder`, `closeposition`, plus full **GTT** (`placegtt`/`modifygtt`/`cancelgtt`/`gttorderbook`) and options-specific order endpoints.
- **Market data:** `quotes`, `multiquotes`, `depth`, `optionchain`, `optiongreeks`/`multioptiongreeks`, `search`, `expiry`, `ticker`, `instruments`, `chart` (OHLCV), `intervals`, `history`, `syntheticfuture`.
- **Account:** `funds`, `margin`, `orderbook`, `tradebook`, `positionbook`, `openposition`, `holdings`, `orderstatus`, `ping`.
- **Utility:** `analyzer`, `pnl`, `telegram`/`whatsapp` (send), `market/holidays`, `market/timings`.
- **Standard symbol format** across all brokers: equity (`SBIN`), futures (`BANKNIFTY24APR24FUT`), options (`NIFTY28MAR2420800CE`); 14+ exchange codes (NSE/BSE/NFO/BFO/CDS/BCD/MCX/NCDEX/`*_INDEX`/`GLOBAL_INDEX`).

### 3.2 Python Strategy Host (`/python`)

- Paste/edit strategies in a browser **CodeMirror** editor.
- Each strategy runs in its **own subprocess** (process isolation; SIGTERM→5s→SIGKILL on stop; cross-platform Windows `.bat` / POSIX wrappers).
- **Schedule** on IST cron times via APScheduler (jobs persisted to DB, survive restarts).
- **Live logs** stream to the UI over SSE / SocketIO.
- Strategies auto-restore after login once the master contract is downloaded.

### 3.3 Flow — No-Code Builder (`/flow`)

- Drag-and-drop **node graph** (market data → indicators → conditions → delays → order actions) using XY Flow on the frontend.
- Definitions stored as JSON (`flow_db.py`); `flow_executor_service.py` (~2.3K LOC) interprets the graph with cycle detection and per-workflow locks.
- Three trigger types: **webhooks** (HMAC token/secret), **scheduled** (APScheduler), and **live price monitors** (5-second polling).
- JSON import/export of whole flows.

### 3.4 Options Trading Suite (`/tools`, 12 tools)

Option Chain (with ITM/ATM/OTM + Greeks), Option Greeks (Δ/Γ/Θ/Vega via `opengreeks`), IV Smile, IV Chart (term structure), Vol Surface (3D strike×expiry), Max Pain, GEX (gamma exposure), OI Tracker, OI Profile, Multi-strike OI, Straddle Chart, Custom Straddle, and a multi-leg Strategy Builder + portfolio view. Each tool is a blueprint + service pair that batch-fetches quotes and renders with Plotly / Lightweight-Charts.

### 3.5 Sandbox / Analyzer (shared by all surfaces)

- A fully **isolated virtual exchange** (`db/sandbox.db`) seeded with **₹1 Crore** configurable capital.
- Realistic **margin + leverage** (configurable per product: MIS/CNC/futures/options-buy/options-sell).
- Background **execution thread** fills pending orders when live quotes cross, and a **square-off thread** auto-closes MIS positions at exchange timings (NSE/BSE 15:20, CDS 17:30, MCX 23:55, NCDEX 16:50 IST).
- Toggle "Analyze mode" to route any order (API/Flow/Python) to the sandbox instead of the live broker. Request/response inspection at `/analyzer`.

### 3.6 Action Center (order approval)

- **Auto mode:** orders execute immediately.
- **Semi-auto mode:** orders queue as `PendingOrder` rows for manual approve/reject before hitting the broker (managed-account workflow). Status checks and GTT modify/cancel are never queued (stale-risk).

### 3.7 Real-time market data

The 3-layer WebSocket pipeline (see [diagrams §4](02-architecture-diagrams.md#4-market-data-websocket--the-3-layer-feed)): broker WS adapters → ZeroMQ → unified proxy on `:8765`, with O(1) tick routing, per-symbol throttling, automatic multi-connection sharding up to 3000 symbols, and stale-token recovery.

### 3.8 Integrations & notifications

- **Telegram & WhatsApp** bots — event-driven alerts on order/position changes, plus command bots (`/funds`, `/positions`, `/orders`) and chart rendering (Kaleido/Chromium).
- **MCP** — control OpenAlgo from Claude/Cursor/Windsurf. Local stdio server (`mcp/mcpserver.py`) plus remote **HTTP/SSE transport** (`mcp_http.py`) with **OAuth2** (`mcp_oauth.py`), scoped read/write tokens, and per-token rate limits.
- **Webhook sources:** TradingView, ChartInk, GoCharting (API key in body/URL — they can't set custom headers, an accepted trade-off).

### 3.9 Historify (historical data)

DuckDB-backed (`historify.duckdb`) historical OHLCV storage with a scheduled backfill service (`historify_scheduler_service.py`) sharing the APScheduler instance. Charting UI under `/historify`.

### 3.10 Observability & ops

- **Traffic logging** at the WSGI layer (every request → `logs.db`), **latency** and **health** monitoring as background threads with `/health/*` probes (K8s/ELB-friendly).
- **Centralized logging** (`utils/logging.py`): colored console, daily-rotated files, and a structured `log/errors.jsonl` (ERROR+ only, with request context + traceback) — the first place to look when debugging.
- **Security surface:** IP ban list + audit (`/security`), per-IP rate limiting, CSP/CSRF/CORS, encrypted broker tokens at rest.

---

## 4. Who uses which surface

| Persona | Primary surface |
| --- | --- |
| TradingView/ChartInk alert user | Webhook → `/api/v1/placeorder` |
| Quant writing Python | Python Strategy Host or the `openalgo` SDK → `/api/v1` |
| Non-coder building rules | Flow `/flow` |
| Options trader | `/tools` suite + Option Chain |
| Managed-account operator | Action Center semi-auto mode |
| AI-assisted trader | MCP from Claude/Cursor |
| Anyone testing | Sandbox / Analyzer |

---

## 5. Deployment options (all documented)

Ubuntu bare-metal (with Caddy + systemd, `install.sh`), single Docker (`install-docker.sh`), multi-instance Docker with custom SSL (`install-multi.sh` / `…custom-ssl.sh`), Windows/macOS dev, and cloud. All official installers auto-generate `APP_KEY` and `API_KEY_PEPPER` via `secrets.token_hex(32)`. Multi-arch Docker images (amd64 + arm64) are built natively by CI. Production servers need **no Node.js** — CI force-commits `frontend/dist/` to `main`, so `git pull` ships the latest UI.

See [03-deep-scan-findings.md](03-deep-scan-findings.md) for the full inventory and [04-strengths-and-improvements.md](04-strengths-and-improvements.md) for the assessment.
