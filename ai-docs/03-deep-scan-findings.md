# OpenAlgo — Deep-Scan Findings (Inventory)

The detailed evidence behind the diagrams. Organized by subsystem with `file:line` anchors where useful.

---

## A. Backend core (`app.py`, middleware, API)

### Startup & middleware
- `app.py` — `create_app()` factory; background `setup_environment()`; parallel DB init on a 15-worker `ThreadPoolExecutor` gated by a `db_ready` event (`app.py:605-660`); startup banner.
- WSGI stack (registered `app.py:319-323`): **SecurityMiddleware** (`security_middleware.py`, IP-ban → 403) wrapped by **TrafficLoggerMiddleware** (`traffic_logger.py`, request/response logging to SQLite via a single-threaded executor). CSP/headers in `csp.py` `after_request`.
- `before_request`: `wait_for_db_ready()` and `check_session_expiry()` (note: explicitly skips `/api/` paths — see improvement #7).
- `teardown_appcontext` removes ~18 scoped sessions every request (FD-leak prevention).
- Extensions: `extensions.py` (SocketIO, threading mode, `ping_timeout=60s`), `limiter.py` (Flask-Limiter, in-memory moving-window, keyed by IP), `cors.py` (conditional on `CORS_ENABLED`), `csp.py` (env-driven, per-response override for OAuth).

### Authentication
- API-key verification: `database/auth_db.py:834-920` — Argon2 + pepper, two-tier cache (valid 1h TTL, invalid 5m TTL), **revocation re-checked even on cache hit** (`auth_db.py:960-968`).
- CSRF (`app.py:144-232`): surgical exemptions for `/api/v1`, webhooks, health; OAuth consent form keeps CSRF.

### `restx_api/` — 49 modules, all under `/api/v1/`
Grouped: **Orders (9)** place/smart/basket/split/modify/cancel/cancelall/closeposition/GTT · **Market data (11)** quotes/multiquotes/depth/optionchain/optiongreeks/search/expiry/ticker/instruments/chart/intervals · **Account & positions (9)** orderbook/tradebook/positionbook/openposition/holdings/funds/margin/orderstatus/ping · **Options (4+)** optionsymbol/optionsorder/optionsmultiorder/syntheticfuture · **Utility (6)** analyzer/pnl/telegram/whatsapp/market-holidays/market-timings. Validation via Marshmallow (`schemas.py`, `data_schemas.py`).

### `blueprints/` — 48 modules
- **Auth/session:** `auth.py`, `apikey.py`, `brlogin.py`
- **Core UI:** `core.py`, `dashboard.py`, `react_app.py`, `platforms.py`
- **Orders/positions:** `orders.py`
- **Automation:** `strategy.py`, `flow.py`, `python_strategy.py`, `chartink.py`, `tv_json.py`, `gc_json.py`
- **Analytics/options:** `analyzer.py`, `pnltracker.py`, `gex.py`, `ivsmile.py`, `ivchart.py`, `oiprofile.py`, `oitracker.py`, `straddle_chart.py`, `custom_straddle.py`, `vol_surface.py`, `strategy_chart.py`, `strategy_portfolio.py`
- **Data:** `historify.py`, `search.py`, `leverage.py`
- **System:** `settings.py`, `security.py`, `traffic.py`, `latency.py`, `health.py`, `logging.py`, `log.py`, `admin.py`, `system_permissions.py`, `sandbox.py`
- **Notifications:** `telegram.py`, `whatsapp.py`
- **MCP:** `mcp_http.py`, `mcp_oauth.py`
- **Dev tools:** `playground.py`, `websocket_example.py`

---

## B. Broker layer & WebSocket (`broker/`, `websocket_proxy/`)

### 32 brokers (+`dhan_sandbox`)
aliceblue, angel, arrow, compositedge, definedge, **deltaexchange (crypto)**, dhan, **dhan_sandbox**, firstock, fivepaisa, **fivepaisaxts**, flattrade, fyers, groww, ibulls, iifl, iiflcapital, indmoney, **jainamxts**, kotak, motilal, mstock, nubra, paytm, pocketful, rmoney, samco, shoonya, tradejini, upstox, wisdom, zebu, zerodha. (XTS-protocol variants flagged; all but deltaexchange target Indian markets.)

### Plugin loading
- `utils/plugin_loader.py:17-54` loads `plugin.json` capabilities at startup; `:65-129` `_LazyBrokerAuthDict` defers broker-SDK imports until first use (saves ~3.5s cold boot). `VALID_BROKERS` env gates which brokers appear.

### WebSocket proxy (`websocket_proxy/`)
- `server.py:34-1502` `WebSocketProxy` — client auth (15s grace), subscription index `(symbol,exchange,mode)→{client_ids}` for O(1) routing, ~50ms throttle, stale-token recovery (`:832-967`).
- `base_adapter.py:97-592` `BaseBrokerWebSocketAdapter` — ZeroMQ/FD lifecycle, `socket.close(linger=0)`, `_bound_ports` tracking, shared ZMQ context with refcount, `__del__` guard.
- `connection_manager.py:224-450+` `ConnectionPool` — auto-shards symbols across adapters up to `MAX_WEBSOCKET_CONNECTIONS`; `SharedZmqPublisher` singleton.
- `broker_factory.py:79-130` adapter factory + optional pooling wrapper.
- Shared HTTP: `utils/httpx_client.py:19-75` — one pooled HTTP/2 client per broker session.

### Reference adapters
`broker/zerodha/streaming/zerodha_adapter.py` (~788 LOC), `broker/angel/…` (~670), `broker/dhan/…` (~700+). SymToken model + composite index in `broker/*/database/master_contract_db.py`.

---

## C. Services & feature subsystems (`services/`, 68 modules)

- **Order routing (13):** place/smart/options/GTT order services, modify/cancel/cancelall/closeposition, basket/split/options-multi, `order_router_service.py` (auto vs semi-auto gate, `:37-77`).
- **Positions/holdings (5):** openposition, positionbook, holdings, tradebook, close_position.
- **Market data (7):** quotes, market_data (singleton), instruments, symbol, option_symbol (strike cache), search, depth.
- **Funds/margin (3)** · **GTT (4)** · **Data/history (4):** historify (DuckDB), history (rate-limited), market_calendar, intervals.
- **Sandbox:** `sandbox_service.py` bridge + `sandbox/` engine (`order_manager`, `fund_manager`, `position_manager`, `execution_thread`, `squareoff_thread`, `execution_engine`). DB: `database/sandbox_db.py`.
- **Flow (3):** `flow_executor_service.py` (~2.3K LOC, cycle detection, per-workflow locks), `flow_scheduler_service.py` (APScheduler singleton, persisted jobstore, IST), `flow_price_monitor_service.py` (5s polling).
- **Options analytics (11):** option_chain, option_greeks, iv_smile, iv_chart, vol_surface, gex, oi_tracker, oi_profile, multi_strike_oi, straddle_chart, custom_straddle.
- **Notifications (4+):** `telegram_bot_service.py` (~2.6K LOC, async/sync bridge, SDK client cache), `telegram_alert_service`, `whatsapp_*`. ⚠️ **3 telegram variants exist** (`telegram_bot_service.py`, `_fixed.py`, `_v2.py`).
- **Scheduling/background (3):** historify_scheduler, broker_keepalive, pending_order_execution.
- **MCP:** `blueprints/mcp_http.py` (JSON-RPC: initialize/tools-list/tools-call), `mcp_oauth.py` + `database/oauth_db.py` (Argon2-hashed tokens, scopes, 60/min read · 50/min write), `mcp/mcpserver.py` (FastMCP, dual stdio/HTTP boot), audit at `log/mcp.jsonl`.
- **Action Center:** `database/action_center_db.py` (`PendingOrder` w/ IST timestamps, approver tracking), `services/action_center_service.py`.
- **Event bus:** `utils/event_bus.py` (pub/sub on thread pool); SocketIO emits `order_update`, `analyzer_update`, `cache_loaded`, `position_update`; master-contract hook `database/master_contract_cache_hook.py:14-55`.

---

## D. Databases (6)

| DB | File | Engine | Holds |
| --- | --- | --- | --- |
| openalgo.db | `database/auth_db.py` et al. | SQLite/NullPool | users, API keys, tokens, orders, strategies, flows, OAuth, action center |
| logs.db | `database/apilog_db.py` | SQLite | API + traffic logs |
| latency.db | `database/latency_db.py` | SQLite | latency metrics |
| health.db | `database/health_db.py` | SQLite | health monitoring |
| sandbox.db | `database/sandbox_db.py` | SQLite (isolated) | virtual orders/positions/funds/holdings |
| historify.duckdb | `database/historify_db.py` | DuckDB | historical OHLCV |

All SQLite via `database.engine_factory.create_db_engine()` with **NullPool** (StaticPool banned). WAL mode + `synchronous=NORMAL`.

---

## E. Frontend (`frontend/`, React 19 + TS + Vite)

- **Stack:** React 19.2, TypeScript 5.9, Vite 8, Tailwind 4.3 + shadcn/ui (36 wrappers), TanStack Query v5, Zustand stores, Socket.IO client 4.8, Plotly 3.3 / Lightweight-Charts 5.1 / XY Flow 12, CodeMirror, Biome lint/format. Node ≥20.20/22.22/24.13.
- **Structure:** `src/pages/` (50+ lazy routes), `src/components/` (~20K LOC, incl. `ui/`, `trading/`, `option-chain/`, `strategy-builder/`, `flow/`), `src/hooks/` (15), `src/stores/`, `src/api/` (22 typed clients), `src/lib/MarketDataManager.ts`, `src/contexts/MarketDataContext.tsx`.
- **Three comms channels:** REST via Axios+TanStack (CSRF token from `/auth/csrf-token`; 401→login interceptor); Socket.IO for order/analyzer/cache events (debounced refetch); raw WS to `:8765` via `MarketDataManager` (ref-counted subs, REST fallback after 3 failures, pause on hidden tab).
- **Routes** map 1:1 to backend surfaces: dashboard/orderbook/tradebook/positions/holdings, tools/* (optionchain, ivchart, oitracker, maxpain, straddle, volsurface, gex, ivsmile, oiprofile, strategybuilder), sandbox, analyzer, strategy/*, python/*, chartink/*, flow/*, admin/*, logs/*, telegram/*, whatsapp, leverage, historify/*, health.
- **Testing:** Vitest (jsdom, v8 coverage) + Playwright (5 browser projects, axe-core a11y). Only **4 unit test files** + 4 E2E specs.
- **Served by** `blueprints/react_app.py` from CI-built `frontend/dist/` (Brotli/gzip variants; `index.html` no-cache).

> Correction to one scan note: the `:8765` endpoint is the **Python** `websocket_proxy/server.py`, not a "C++ kernel."

---

## F. Docs, CI/CD, deployment, tests

### Docs (~237 md files)
- Top-level: `README.md`, `CLAUDE.md` (27KB dev guide), `CONTRIBUTING.md` (33KB), `INSTALL.md` (⚠️ stale — references Py 3.10/3.11), `SECURITY.md`, `DOCKER_README.md`.
- `docs/`: `websocket-architecture.md`, `scanner-architecture.md`, `broker-integration-guide.md` (50KB), `architecture-diagram.png`, plus subdirs `design/` (56), `userguide/` (36), `prd/` (25), `installation-guidelines/`, `releases/` (9), `migration/`, `test/`.
- `docs/migration/` — **Flask→FastAPI migration tracker** (CSV, ~50 items, 4 phases P0–P4, 15 tracked risks). **Status: Not Started** as of 13-Jun-2026.

### CI/CD (`.github/workflows/`)
- `ci.yml` (356 lines): Ruff lint/format (continue-on-error), **5 CI-safe backend tests**, Biome lint, Vite build + Vitest + Playwright (Chromium), Bandit + pip-audit (continue-on-error), **`commit-dist`** job force-commits `frontend/dist/` to main, native multi-arch Docker (amd64 + arm64-native, Kaleido smoke test), Trivy scan.
- `security.yml`: weekly Bandit + pip-audit (tolerant).

### Deployment
- Multi-stage `Dockerfile` (python-builder → node-builder → 3.12-slim runtime, IST tz, Chromium+fonts for Kaleido, appuser UID 1000, healthcheck on `/auth/check-setup`).
- `docker-compose.yaml` (ports 5000/8765, 5 named volumes, `shm_size` for scipy/numba, `restart: unless-stopped`).
- `install/` (14 scripts): bare-metal, single/multi Docker, custom SSL, domain change, remote MCP, update.
- `pyproject.toml`: Py≥3.12, Flask 3.1.3, SQLAlchemy 2.0.49, pandas/numpy, Plotly 6.6, kaleido, pyzmq 27, argon2-cffi, APScheduler, python-telegram-bot 22; dev group: pytest, bandit, pip-audit, detect-secrets; Ruff config (E/F/W/I/B/C4/UP, line 100).

### Tests — honest numbers
- **Backend:** 61 `test_*.py` files, ~194 `test_*` functions, but only **5 run in CI** (log location, navigation, python editor, rate limits, logout CSRF). The rest are manual/local (need live broker creds, market hours, external services). Rough **~21% file coverage**.
- **Frontend:** 4 unit test files + 4 Playwright specs, **no coverage gate**.
- CLAUDE.md itself states: *"Most testing is currently manual."*
- **Zero automated contract tests** for the 49 REST namespaces, the order lifecycle, the WebSocket feed, or the broker integrations — the financial-critical paths.

---

## G. Notable cross-cutting observations

- **FD discipline is a first-class concern** — CLAUDE.md dedicates a section to it; NullPool, shared httpx client, ZMQ `linger=0`, teardown session cleanup all reflect hard-won lessons about a long-running single-worker process.
- **Eventlet constraint is pervasive** — anything needing async runs on a separate OS thread (see `telegram_bot_service._render_plotly_png`).
- **Two version numbers** (platform vs SDK pin) are independent — easy to confuse.
- **A real modernization plan exists** (Flask→FastAPI) but is gated behind a test safety-net that does not yet exist.
