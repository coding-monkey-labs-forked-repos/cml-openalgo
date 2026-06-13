# OpenAlgo — Project Deep-Scan & Diagrammatic Reference

> Generated 2026-06-13 from a full code + docs scan of the repository at platform version **`2.0.1.3`**.
> Five parallel exploration agents swept the backend core, broker/WebSocket layer, services & feature subsystems, the React frontend, and docs/CI/deployment/tests. This folder is the synthesized result.

## What is OpenAlgo?

OpenAlgo is a **self-hosted, single-user algorithmic trading platform** for the Indian markets (plus crypto via Delta Exchange). It is **four products in one instance**, all sharing one broker session and one market-data feed:

| Surface | Route | Purpose |
| --- | --- | --- |
| **Unified Broker API** | `/api/v1/` | One REST API in front of 30+ brokers (TradingView, Amibroker, ChartInk, Excel, Python, MCP) |
| **Python Strategy Host** | `/python` | In-browser editor; schedule scripts on IST times; subprocess-isolated runs with live logs |
| **Flow (No-Code Builder)** | `/flow` | Drag-and-drop node graph: data → indicators → conditions → orders |
| **Options Trading Suite** | `/tools` | 12 analytics tools (Option Chain, IV Smile, Max Pain, Vol Surface, GEX, OI Tracker, Straddle…) |

All surfaces share a **Sandbox engine** (₹1 Crore virtual capital, exchange-aligned auto square-off) and **Telegram/WhatsApp** alerts.

## How to read this folder

| File | Contents |
| --- | --- |
| [01-overview-and-capabilities.md](01-overview-and-capabilities.md) | The 30,000-ft view + an exhaustive catalogue of **what the platform can do** |
| [02-architecture-diagrams.md](02-architecture-diagrams.md) | **All the diagrams** — system context, request pipeline, WebSocket 3-layer feed, broker plugin contract, subsystem/feature map, database map, frontend comms |
| [03-deep-scan-findings.md](03-deep-scan-findings.md) | The detailed inventory behind the diagrams — blueprints, REST endpoints, services, brokers, frontend routes, tests/CI/deploy, with `file:line` anchors |
| [04-strengths-and-improvements.md](04-strengths-and-improvements.md) | **10 things this project does well** and **10 things to improve**, each evidence-backed |

## Repository scale (measured)

| Metric | Count |
| --- | --- |
| Python files (excl. `.venv`, `frontend`) | ~920 |
| Broker integrations (`broker/*/`) | 34 dirs (32 brokers + `dhan_sandbox` + `__init__`) |
| Flask blueprints (`blueprints/*.py`) | 48 |
| REST API modules (`restx_api/*.py`) | 49 |
| Service modules (`services/*.py`) | 68 |
| Databases | 6 (5 SQLite + 1 DuckDB) |
| React routes / pages | 50+ |
| Docs (`docs/**/*.md`) | ~237 markdown files |
| Backend test files / functions | 61 files / ~194 `test_*` |
| Frontend test files | 4 (+ Playwright E2E) |

## TL;DR verdict

A **remarkably feature-complete, well-documented, security-conscious** trading platform with a modern React 19 SPA and a genuinely clever WebSocket/broker abstraction. Its two largest risks are **thin automated test coverage on the financial-critical paths** and **massive duplication across 32 broker adapters** (~29K LOC). See [04-strengths-and-improvements.md](04-strengths-and-improvements.md).
