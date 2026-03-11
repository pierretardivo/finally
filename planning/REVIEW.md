# FinAlly Project — Comprehensive Code Review

*Review date: 2026-03-09*

---

## Executive Summary

The project is approximately **10–15% complete**. The market data subsystem is the only implemented component and it is high quality — clean architecture, thorough tests, and faithful spec alignment. Everything else specified in PLAN.md (FastAPI app, database, portfolio/trade logic, LLM chat, the entire frontend, Docker, scripts, E2E tests) does not yet exist.

---

## 1. What Has Been Built

### Market Data Subsystem (`backend/app/market/`)

Fully implemented and well-tested. Matches PLAN.md Sections 6 and 12.

**Files:**
- `models.py` — `PriceUpdate` dataclass (ticker, price, prev_price, timestamp, direction)
- `cache.py` — Thread-safe in-memory price cache using `threading.RLock`
- `interface.py` — `MarketDataSource` ABC defining `start()`, `stop()`, `get_prices()`
- `simulator.py` — GBM-based simulator with correlated moves, random events, `SimulatorDataSource`
- `massive_client.py` — Massive (Polygon.io) REST polling client, `MassiveDataSource`
- `factory.py` — `create_data_source()` factory driven by `MASSIVE_API_KEY` env var
- `stream.py` — SSE streaming router (`GET /api/stream/prices`), server-sent event formatting
- `seed_prices.py` — Seed prices and GBM parameters for all 10 default tickers

**Test coverage (73 tests across 6 modules):**
- `test_simulator.py` — GBM math, price generation, correlation, random events
- `test_massive_client.py` — REST response parsing, error handling
- `test_cache.py` — Thread-safety, CRUD operations
- `test_interface.py` — ABC contract conformance for both implementations
- `test_stream.py` — SSE event format, endpoint behavior
- `test_factory.py` — Env-var-driven selection logic

**Demo:**
- `market_data_demo.py` — Rich terminal demo showing live price updates; confirmed working

---

## 2. What Has NOT Been Built

The following components from PLAN.md are entirely absent:

| Component | Plan Section | Status |
|---|---|---|
| FastAPI main application (`main.py`) | §3, §8 | **Missing** |
| Database init, schema, seed | §7 | **Missing** |
| Portfolio API (`/api/portfolio*`) | §8 | **Missing** |
| Trade execution logic | §8, §9 | **Missing** |
| Watchlist API (`/api/watchlist*`) | §8 | **Missing** |
| Chat API + LLM integration | §8, §9 | **Missing** |
| Health endpoint (`/api/health`) | §8 | **Missing** |
| Portfolio snapshot background task | §7 | **Missing** |
| Static file serving for frontend | §3, §11 | **Missing** |
| Frontend (Next.js, all components) | §10 | **Missing** |
| Dockerfile (multi-stage build) | §11 | **Missing** |
| `docker-compose.yml` | §11 | **Missing** |
| Start/stop scripts (`scripts/`) | §11 | **Missing** |
| E2E tests (`test/`) | §12 | **Missing** |
| `.env.example` | §5 | **Missing** |

---

## 3. Issues in the Existing Code

### 3.1 Bug — Inconsistent ticker case normalization

`MassiveDataSource` normalizes tickers to uppercase on receipt, but `SimulatorDataSource` does not normalize ticker case at all. If a ticker is added to the watchlist in lowercase (e.g., `"aapl"`), the simulator will accept it as a separate key from `"AAPL"`, causing silent mismatches. Both implementations should normalize to uppercase.

**Location:** `backend/app/market/simulator.py`, `backend/app/market/massive_client.py`

### 3.2 Fragile pattern — Module-level router mutation in `stream.py`

`create_stream_router()` creates a `APIRouter` and immediately mounts an endpoint on it. If called more than once (e.g., in tests), duplicate routes will be registered on the same router object. The router should be created outside the factory function, or the function should always return a fresh router instance.

**Location:** `backend/app/market/stream.py`

### 3.3 Documentation error — `backend/README.md`

`backend/README.md` documents `uv sync --dev` for installing dev dependencies. The correct command for `pyproject.toml` optional extras is `uv sync --extra dev` (or `uv sync --all-extras`). The `--dev` flag does not exist in current `uv`.

**Location:** `backend/README.md`

### 3.4 Hard dependency on `massive` package

`pyproject.toml` lists the `massive` package as a regular (non-optional) dependency. Most users will never use Massive API (they'll use the simulator). This adds unnecessary install weight and potential import errors for users without the key. Consider making it an optional extra: `uv add massive --optional massive`.

**Location:** `backend/pyproject.toml`

### 3.5 No FastAPI entry point

There is no `backend/app/main.py` or equivalent. The SSE router exists but cannot be started. This is the most critical gap — nothing can run.

---

## 4. Open Questions from PLAN.md §13 That Must Be Resolved Before Building

The doc-review section in PLAN.md identified several open questions. The ones that block parallel frontend/backend development are:

1. **API response shapes** (§8 review note): `/api/portfolio`, `/api/watchlist`, `/api/portfolio/history`, `/api/chat` response JSON shapes are not defined. Frontend and backend agents need a shared contract. **Recommend adding example JSON to PLAN.md before building.**

2. **Zero-quantity positions** (§7 review note): When all shares are sold, should the `positions` row be deleted or set to `quantity=0`? This affects portfolio display logic. **Recommend: delete the row** (cleaner, no dead state).

3. **Daily change %** (§2 review note): No prior-day close exists in simulator mode. **Recommend: show change since simulation start**, clearly labeled.

4. **Chat history on page refresh** (§8 review note): There is no `GET /api/chat` endpoint. Without it, conversation history is lost on refresh. **Recommend: add the endpoint** — it's a simple SELECT from `chat_messages`.

5. **LLM mock response shape** (§9 review note): The exact JSON returned by `LLM_MOCK=true` is unspecified. E2E tests cannot be written without this. **Recommend: define and document the mock response now.**

6. **Conversation history limit** (§9 review note): No cap on messages sent to LLM. **Recommend: last 20 messages.**

---

## 5. Architecture Assessment

### Strengths
- The two-implementation / one-interface pattern for market data is clean and extensible
- Thread-safe cache design is correct for the concurrent SSE use case
- GBM simulator with correlated moves and random events creates realistic-feeling price action
- Test coverage on the completed subsystem is comprehensive

### Concerns for Upcoming Work
- **Single container architecture** is sound but the static-file serving path (Next.js export → FastAPI `StaticFiles`) must be correctly configured in both the Dockerfile and `main.py` or the app won't load
- **SQLite lazy init** pattern is correct but the init code must be called before the first request handler runs, or concurrent first requests could race to create the schema
- **Background tasks** (snapshot recorder + price cache updater) need proper startup/shutdown lifecycle hooks in FastAPI (`lifespan` async context manager, not deprecated `on_event`)
- **LLM structured output** via LiteLLM + OpenRouter requires careful error handling — malformed or incomplete JSON from the model must not crash the chat endpoint

---

## 6. Recommendations for Next Steps

In priority order:

1. **Resolve the open API response shape questions** — add example JSON to PLAN.md so frontend and backend can be built in parallel without mismatch
2. **Build `backend/app/main.py`** — FastAPI app, lifespan hooks, mount SSE router, mount static files
3. **Build the database layer** — schema init, seed, and the 5 table-backed routers
4. **Build the LLM chat endpoint** — using the cerebras skill, with mock mode
5. **Build the frontend** — using the frontend-design skill
6. **Build the Dockerfile and scripts** — tie everything together
7. **Write E2E tests** — using Playwright against the running container

---

## 7. Minor Observations

- `backend/market_data_demo.py` is a development convenience script; it should not be included in the Docker image (add to `.dockerignore`)
- The `.claude/skills/cerebras/SKILL.md` defines `openrouter/openai/gpt-oss-120b` as the model — this matches PLAN.md §9
- `planning/MARKET_DATA_SUMMARY.md` is a good pattern; similar summary docs should be written after each major component is completed so future agents have a fast-path to understanding what was built

---

*Review carried out by the reviewer subagent. All findings are based on static analysis of the codebase — no code was executed.*
