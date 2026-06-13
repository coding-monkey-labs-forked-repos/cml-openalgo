# OpenAlgo — Architecture Diagrams

All diagrams are [Mermaid](https://mermaid.js.org) and render in GitHub, VS Code (with a Mermaid extension), and most Markdown viewers. ASCII fallbacks are included for the two hottest paths.

---

## 1. System Context — who talks to OpenAlgo

```mermaid
graph TB
    subgraph clients["External clients & signal sources"]
        TV["TradingView / ChartInk<br/>GoCharting (webhooks)"]
        AMI["Amibroker / Excel / Python SDK"]
        MCP_C["Claude / Cursor / Windsurf<br/>(MCP)"]
        BROWSER["Browser<br/>(React 19 SPA)"]
        TG["Telegram / WhatsApp"]
    end

    subgraph oa["OpenAlgo — single self-hosted instance (1 user, 1 broker session)"]
        FLASK["Flask app :5000<br/>(Gunicorn + eventlet, -w 1)"]
        WSP["WebSocket Proxy :8765<br/>(market data fan-out)"]
        ZMQ["ZeroMQ bus :5555<br/>(internal pub/sub)"]
    end

    subgraph ext["Broker side"]
        BROKERS["30+ Broker REST + WS APIs<br/>(Zerodha, Dhan, Angel, Upstox, …)"]
    end

    TV -->|"REST /api/v1 (apikey in body)"| FLASK
    AMI -->|"REST /api/v1"| FLASK
    MCP_C -->|"MCP over HTTP/SSE + OAuth"| FLASK
    BROWSER -->|"REST + Socket.IO"| FLASK
    BROWSER -->|"market ticks (ws)"| WSP
    FLASK <-->|"orders, funds, history (httpx HTTP/2)"| BROKERS
    FLASK -->|"alerts"| TG
    BROKERS -->|"tick feed (broker ws)"| ZMQ
    ZMQ --> WSP
    WSP --> BROWSER

    classDef oa fill:#e3f2fd,stroke:#1565c0,stroke-width:2px;
    class FLASK,WSP,ZMQ oa;
```

**Key fact:** orders/account go through Flask → broker REST; market data is a *separate* pipeline (broker WS → ZeroMQ → proxy → browser) so a slow chart client can never block order placement.

---

## 2. Backend Component Architecture

```mermaid
graph TD
    REQ["Incoming HTTP request"]

    subgraph wsgi["WSGI middleware (outermost first)"]
        TRAF["TrafficLoggerMiddleware<br/>traffic_logger.py — logs method/path/status/duration"]
        SEC["SecurityMiddleware<br/>security_middleware.py — IP ban → 403"]
    end

    subgraph flask["Flask app (app.py)"]
        CSP["CSP / security headers (csp.py after_request)"]
        CSRF["Flask-WTF CSRF (exempt: /api/v1, webhooks)"]
        RL["Flask-Limiter (limiter.py)"]
        ROUTER{"Route dispatch"}
    end

    subgraph surfaces["Route surfaces"]
        REACT["react_app.py → serves frontend/dist/"]
        APIV1["restx_api/ → /api/v1/* (49 modules, Swagger)"]
        BP["blueprints/ → 48 UI + webhook blueprints"]
        MCPH["mcp_http.py / mcp_oauth.py → /mcp, /oauth"]
    end

    SVC["services/ (68 modules)<br/>business logic + orchestration"]
    EVT["Event bus (events/ + subscribers/)<br/>pub/sub on a thread pool"]
    BROKER["broker/ (32 plugins)<br/>dynamically loaded via plugin_loader.py"]
    DB[("6 databases<br/>SQLAlchemy ORM, NullPool")]
    SIO["Flask-SocketIO<br/>order_update / analyzer_update / cache_loaded"]

    REQ --> TRAF --> SEC --> CSP --> CSRF --> RL --> ROUTER
    ROUTER --> REACT
    ROUTER --> APIV1
    ROUTER --> BP
    ROUTER --> MCPH
    APIV1 --> SVC
    BP --> SVC
    MCPH --> SVC
    SVC --> BROKER
    SVC --> DB
    SVC --> EVT
    EVT --> SIO
    EVT -->|"alerts"| BROKER
    BROKER <-->|"httpx HTTP/2 pool"| EXT["Broker APIs"]

    classDef svc fill:#fff3e0,stroke:#e65100;
    class SVC,EVT svc;
```

> Middleware registration is in `app.py:319-323` — security registered first so traffic logging wraps *outside* it. Session cleanup runs in `teardown_appcontext` after the response is sent (removes ~18 scoped sessions to prevent FD leaks).

---

## 3. Request Processing Pipeline (order placement, end-to-end)

```mermaid
sequenceDiagram
    participant C as Client (TradingView / SDK)
    participant MW as WSGI middleware
    participant API as restx_api/place_order.py
    participant SVC as services/place_order_service.py
    participant ROUTE as order_router_service.py
    participant AC as Action Center (semi-auto)
    participant BR as broker/<name>/api/order_api.py
    participant EV as Event bus → SocketIO

    C->>MW: POST /api/v1/placeorder {apikey, symbol, …}
    MW->>MW: IP-ban check, traffic log start
    MW->>API: dispatch (CSRF-exempt)
    API->>API: Marshmallow validate + rate limit (10/s)
    API->>SVC: place_order(data, apikey)
    SVC->>SVC: get_auth_token_broker(apikey)<br/>(Argon2 verify + cache + revocation check)
    SVC->>ROUTE: route order
    alt order_mode == semi-auto
        ROUTE->>AC: queue PendingOrder → status:pending
        AC-->>C: {status:"pending", id}
    else order_mode == auto
        ROUTE->>BR: place_order(payload, token) via httpx
        BR-->>SVC: broker order id / status
        SVC->>EV: publish OrderPlacedEvent
        EV->>EV: SocketIO emit order_update + Telegram alert
        SVC-->>C: {status:"success", orderid}
    end
    MW->>MW: traffic log finish, session teardown
```

If **Analyze/Sandbox mode** is on, the same path is intercepted by `sandbox_service.py` and routed to the virtual exchange instead of the live broker.

---

## 4. Market-Data WebSocket — the 3-layer feed

```mermaid
graph LR
    subgraph L3["Layer 3 — broker native"]
        BWS["Broker WebSocket<br/>(Zerodha/Angel/Dhan …)"]
    end
    subgraph L2["Layer 2 — adapters + ZeroMQ bus"]
        ADP["broker/*/streaming/<name>_adapter.py<br/>normalizes ticks → OpenAlgo format"]
        POOL["ConnectionPool<br/>auto-shards symbols across N adapters"]
        PUB["ZeroMQ PUB :5555<br/>topic = EXCHANGE_SYMBOL_MODE"]
    end
    subgraph L1["Layer 1 — unified proxy"]
        SUB["ZeroMQ SUB"]
        IDX["subscription_index<br/>(symbol,exch,mode) → {client_ids}  O(1)"]
        THR["per-symbol throttle (~50ms)"]
        PROXY["websocket_proxy/server.py :8765"]
    end
    BROWSER["Browser clients<br/>(LTP / Quote / Depth)"]

    BWS --> ADP --> POOL --> PUB --> SUB --> IDX --> THR --> PROXY --> BROWSER
    BROWSER -.->|"auth + subscribe/unsubscribe"| PROXY

    classDef l fill:#e8f5e9,stroke:#2e7d32;
    class ADP,POOL,PUB,SUB,IDX,THR,PROXY l;
```

**Capacity:** `MAX_SYMBOLS_PER_WEBSOCKET` (1000) × `MAX_WEBSOCKET_CONNECTIONS` (3) = **3000 symbols** per broker session, auto-sharded. Modes: `1=LTP`, `2=Quote`, `3=Depth`.

ASCII view of the hot path (tick delivery):

```
broker tick ──► adapter.publish_market_data(topic,data)
                topic = "NSE_RELIANCE_2"
                        │ ZeroMQ multipart [topic][json]
                        ▼
   proxy.zmq_listener():  parse topic → (sym,exch,mode)
                          throttle? (skip if <50ms since last)
                          subscription_index[(sym,exch,mode)] → {client_7, client_12}
                          for each live client: ws.send(data)
```

---

## 5. Broker Plugin Contract (every one of 32 brokers implements this)

```mermaid
graph TD
    PJ["plugin.json<br/>name, supported_exchanges, broker_type, leverage_config"]
    subgraph dir["broker/&lt;name&gt;/"]
        API["api/<br/>• auth_api.py → authenticate_broker()<br/>• order_api.py → place/modify/cancel<br/>• data.py → quotes/depth/history<br/>• funds.py → balance/margin"]
        MAP["mapping/<br/>OpenAlgo format ⇄ broker format<br/>(order_data, transform_data, margin_data)"]
        STREAM["streaming/<br/>&lt;name&gt;_adapter.py : BaseBrokerWebSocketAdapter<br/>initialize/connect/subscribe/unsubscribe/disconnect"]
        MCDB["database/master_contract_db.py<br/>SymToken: symbol⇄brsymbol⇄token + index"]
    end
    LOADER["utils/plugin_loader.py<br/>lazy-discovers plugin.json,<br/>imports adapter/auth only on first use"]

    PJ --> LOADER
    LOADER --> dir
```

**Why this matters:** add a broker by creating one directory that fills this contract + adding it to `VALID_BROKERS`. **Cost:** the contract is copy-implemented per broker, so the 32 adapters total ~29K LOC of near-duplicate code (see improvement #2).

---

## 6. Feature / Subsystem Map — what orchestrates what

```mermaid
graph TD
    APP["app.py — create_app()"]

    subgraph trading["Trading core"]
        ORD["orders blueprint + restx_api"] --> PSVC["place_order_service"]
        PSVC --> ROUTER["order_router_service<br/>(auto vs semi-auto)"]
        ROUTER --> AC["Action Center<br/>action_center_db + service"]
        ROUTER --> SBX["Sandbox engine<br/>sandbox/ + sandbox_service"]
        ROUTER --> LIVE["live broker API"]
    end

    subgraph auto["Automation surfaces"]
        PY["Python Strategy Host<br/>python_strategy.py + subprocess + APScheduler"]
        FLOW["Flow no-code<br/>flow_executor + flow_scheduler + flow_price_monitor"]
        STRAT["Webhook strategies<br/>strategy.py / chartink.py / tv_json.py"]
    end

    subgraph analytics["Options analytics (/tools)"]
        OPT["option_chain, iv_smile, vol_surface, gex,<br/>oi_tracker, oi_profile, straddle, greeks, max-pain"]
    end

    subgraph data["Data & history"]
        HIST["Historify (DuckDB)<br/>historify_service + historify_scheduler"]
        MC["Master contract cache hook<br/>→ socketio cache_loaded"]
    end

    subgraph notify["Notifications & integrations"]
        TG["telegram_bot_service"]
        WA["whatsapp_bot_service"]
        MCPSVC["MCP: mcp_http / mcp_oauth / mcpserver"]
    end

    EVTBUS["Event bus (pub/sub, thread pool)"]
    SIO["Flask-SocketIO → React UI"]

    APP --> trading
    APP --> auto
    APP --> analytics
    APP --> data
    APP --> notify
    PSVC --> EVTBUS
    PY --> PSVC
    FLOW --> PSVC
    STRAT --> PSVC
    OPT --> Q["quotes / multiquotes / optionchain services"]
    EVTBUS --> SIO
    EVTBUS --> TG
    EVTBUS --> WA
    MC --> SIO
```

---

## 7. Database Map (6 isolated stores)

```mermaid
graph LR
    subgraph sqlite["SQLite (NullPool, WAL)"]
        MAIN[("openalgo.db<br/>users, API keys (Argon2),<br/>orders, strategies, flows,<br/>OAuth, action center")]
        LOGS[("logs.db<br/>API + traffic logs")]
        LAT[("latency.db<br/>latency metrics")]
        HLT[("health.db<br/>health monitoring")]
        SAND[("sandbox.db<br/>virtual orders/positions/funds<br/>fully isolated from live")]
    end
    subgraph duck["DuckDB (columnar)"]
        HISTO[("historify.duckdb<br/>historical OHLCV")]
    end

    classDef main fill:#ffebee,stroke:#c62828;
    class MAIN main;
```

> Every SQLite engine is created via `database.engine_factory.create_db_engine()` with **NullPool** (fresh connection per op, closed immediately). `StaticPool` is explicitly banned — it corrupts cursor state under concurrency.

---

## 8. Frontend ⇄ Backend Communication (3 channels)

```mermaid
graph TD
    subgraph fe["React 19 SPA (frontend/)"]
        PAGES["50+ lazy-loaded pages"]
        TQ["TanStack Query (REST cache)"]
        ZUS["Zustand stores (auth, broker, theme, alerts)"]
        MDM["MarketDataManager<br/>(singleton ws, ref-counted subs,<br/>REST fallback after 3 failures)"]
        SIOC["Socket.IO client"]
    end

    REST["Flask /api/v1/* + /auth/*<br/>(Axios + CSRF token)"]
    SIOS["Flask-SocketIO /socket.io<br/>(order/analyzer/cache events)"]
    WSPROX["WebSocket proxy :8765<br/>(LTP / Quote / Depth)"]

    TQ -->|"queries/mutations"| REST
    SIOC -->|"event → refetch (debounced)"| SIOS
    MDM -->|"subscribe symbols"| WSPROX
    PAGES --> TQ
    PAGES --> MDM
    PAGES --> ZUS
    SIOS -.->|"push"| SIOC
    WSPROX -.->|"ticks"| MDM

    classDef fe fill:#ede7f6,stroke:#4527a0;
    class PAGES,TQ,ZUS,MDM,SIOC fe;
```

Served in production by `blueprints/react_app.py` from the CI-built `frontend/dist/` (Brotli/gzip pre-compressed; `index.html` no-cache, hashed assets cached 1 year).

---

## 9. The four product surfaces at a glance

```mermaid
mindmap
  root((OpenAlgo<br/>2.0.1.3))
    Unified Broker API
      /api/v1 — 49 endpoints
      orders / smart / basket / split / GTT
      quotes / depth / option chain / greeks
      funds / margin / positions / holdings
      history / intervals / search
      32 brokers behind one schema
    Python Strategy Host
      in-browser CodeMirror editor
      subprocess isolation
      APScheduler IST cron
      live SSE / SocketIO logs
    Flow No-Code Builder
      node graph (XY Flow)
      conditions + indicators + actions
      webhook + scheduled + price triggers
      JSON import/export
    Options Trading Suite
      Option Chain / Greeks
      IV Smile / IV Chart / Vol Surface
      Max Pain / GEX
      OI Tracker / OI Profile
      Straddle / Custom Straddle / Strategy Builder
    Shared services
      Sandbox (1 Cr virtual capital)
      Action Center (order approval)
      Telegram / WhatsApp alerts
      MCP (Claude / Cursor)
      WebSocket market feed
      Historify (DuckDB)
```
