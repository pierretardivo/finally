# Market Data Backend — Design Document

Comprehensive design of the FinAlly market data subsystem as implemented in `backend/app/market/`. This document describes the architecture, every module with code snippets, data flows, and integration points for downstream consumers (SSE streaming, portfolio valuation, trade execution).

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Module Map](#2-module-map)
3. [Data Model — `models.py`](#3-data-model)
4. [Price Cache — `cache.py`](#4-price-cache)
5. [Abstract Interface — `interface.py`](#5-abstract-interface)
6. [Seed Prices & Ticker Parameters — `seed_prices.py`](#6-seed-prices--ticker-parameters)
7. [GBM Simulator — `simulator.py`](#7-gbm-simulator)
8. [Massive API Client — `massive_client.py`](#8-massive-api-client)
9. [Factory — `factory.py`](#9-factory)
10. [SSE Streaming Endpoint — `stream.py`](#10-sse-streaming-endpoint)
11. [FastAPI Lifecycle Integration](#11-fastapi-lifecycle-integration)
12. [Watchlist Coordination](#12-watchlist-coordination)
13. [Data Flow Diagrams](#13-data-flow-diagrams)
14. [Error Handling & Edge Cases](#14-error-handling--edge-cases)
15. [Testing Strategy](#15-testing-strategy)
16. [Configuration Summary](#16-configuration-summary)

---

## 1. Architecture Overview

The market data subsystem follows the **Strategy pattern**: two interchangeable data sources (simulator and Massive API) implement a common abstract interface. Both write to a shared, thread-safe in-memory `PriceCache`. All downstream consumers (SSE streaming, portfolio valuation, trade execution) read exclusively from the cache — they never interact with the data source directly.

```
┌──────────────────────────────────────────────────────────────┐
│                     Market Data Subsystem                     │
│                                                              │
│   ┌────────────────────┐    ┌────────────────────┐           │
│   │ SimulatorDataSource│    │ MassiveDataSource   │           │
│   │   (GBM engine)     │    │ (Polygon.io REST)   │           │
│   │  updates @ 500ms   │    │  polls @ 15s        │           │
│   └────────┬───────────┘    └────────┬───────────┘           │
│            │ implements               │ implements            │
│            ▼                          ▼                       │
│   ┌──────────────────────────────────────────┐               │
│   │         MarketDataSource (ABC)            │               │
│   │  start() / stop() / add / remove / get   │               │
│   └──────────────────┬───────────────────────┘               │
│                      │ writes to                              │
│                      ▼                                        │
│   ┌──────────────────────────────────────────┐               │
│   │            PriceCache                     │               │
│   │   Thread-safe in-memory dict              │               │
│   │   {ticker → PriceUpdate}                  │               │
│   │   Monotonic version counter               │               │
│   └──────┬──────────┬──────────┬─────────────┘               │
│          │          │          │                              │
│          ▼          ▼          ▼                              │
│      SSE stream  Portfolio  Trade                             │
│      endpoint    valuation  execution                         │
│                                                              │
│   ┌──────────────────────────────────────────┐               │
│   │  create_market_data_source(cache)         │               │
│   │  Factory: reads MASSIVE_API_KEY env var   │               │
│   └──────────────────────────────────────────┘               │
└──────────────────────────────────────────────────────────────┘
```

### Key Design Decisions

| Decision | Rationale |
|---|---|
| Strategy pattern (ABC + two implementations) | Downstream code is source-agnostic; easy to test with either source |
| PriceCache as single point of truth | Decouples producers from consumers; producers write, consumers read |
| Thread-safe cache with `threading.Lock` | SSE generator runs in async context but cache may be accessed from thread pool |
| Version counter on cache | SSE endpoint skips sending data when nothing has changed — avoids stale-price flash animations |
| Factory driven by env var | Zero-config default (simulator); real data opt-in via `MASSIVE_API_KEY` |
| Frozen dataclass for PriceUpdate | Immutable snapshots prevent accidental mutation; safe to share across threads |

---

## 2. Module Map

```
backend/app/market/
├── __init__.py           # Public API re-exports
├── models.py             # PriceUpdate dataclass
├── cache.py              # PriceCache (thread-safe store)
├── interface.py          # MarketDataSource ABC
├── seed_prices.py        # Seed prices, GBM params, correlation groups
├── simulator.py          # GBMSimulator + SimulatorDataSource
├── massive_client.py     # MassiveDataSource (Polygon.io poller)
├── factory.py            # create_market_data_source() factory
└── stream.py             # SSE endpoint (GET /api/stream/prices)
```

Public API (via `__init__.py`):

```python
from app.market import (
    PriceUpdate,           # Immutable price snapshot
    PriceCache,            # Thread-safe in-memory store
    MarketDataSource,      # Abstract interface
    create_market_data_source,  # Factory
    create_stream_router,  # FastAPI SSE router
)
```

---

## 3. Data Model — `models.py`

A single frozen dataclass represents every price update flowing through the system.

```python
@dataclass(frozen=True, slots=True)
class PriceUpdate:
    ticker: str
    price: float
    previous_price: float
    timestamp: float = field(default_factory=time.time)  # Unix seconds

    @property
    def change(self) -> float:
        """Absolute price change from previous update."""
        return round(self.price - self.previous_price, 4)

    @property
    def change_percent(self) -> float:
        """Percentage change from previous update."""
        if self.previous_price == 0:
            return 0.0
        return round((self.price - self.previous_price) / self.previous_price * 100, 4)

    @property
    def direction(self) -> str:
        """'up', 'down', or 'flat'."""
        if self.price > self.previous_price:
            return "up"
        elif self.price < self.previous_price:
            return "down"
        return "flat"

    def to_dict(self) -> dict:
        """Serialize for JSON / SSE transmission."""
        return {
            "ticker": self.ticker,
            "price": self.price,
            "previous_price": self.previous_price,
            "timestamp": self.timestamp,
            "change": self.change,
            "change_percent": self.change_percent,
            "direction": self.direction,
        }
```

### Design Notes

- **`frozen=True`**: Immutable once created. Safe to read from multiple coroutines without copying.
- **`slots=True`**: Memory-efficient; no `__dict__` per instance.
- **`previous_price`**: Computed by the `PriceCache` on each update — the first update for a ticker sets `previous_price == price` (direction = "flat").
- **`direction`**: Used by the frontend to trigger green/red CSS flash animations.
- **`to_dict()`**: Used directly in SSE event serialization. Includes computed properties so the frontend doesn't need to recalculate.

### Example SSE Payload (single ticker)

```json
{
  "ticker": "AAPL",
  "price": 191.23,
  "previous_price": 190.87,
  "timestamp": 1710172800.5,
  "change": 0.36,
  "change_percent": 0.1886,
  "direction": "up"
}
```

---

## 4. Price Cache — `cache.py`

The `PriceCache` is the single point of truth for current prices. One producer writes (simulator or Massive poller), multiple consumers read (SSE, portfolio, trade execution).

```python
class PriceCache:
    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._lock = Lock()
        self._version: int = 0  # Bumped on every update

    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
        """Record a new price. Automatically tracks previous_price."""
        with self._lock:
            ts = timestamp or time.time()
            prev = self._prices.get(ticker)
            previous_price = prev.price if prev else price
            update = PriceUpdate(
                ticker=ticker,
                price=round(price, 2),
                previous_price=round(previous_price, 2),
                timestamp=ts,
            )
            self._prices[ticker] = update
            self._version += 1
            return update

    def get(self, ticker: str) -> PriceUpdate | None: ...
    def get_all(self) -> dict[str, PriceUpdate]: ...
    def get_price(self, ticker: str) -> float | None: ...
    def remove(self, ticker: str) -> None: ...

    @property
    def version(self) -> int:
        """Monotonic counter. SSE uses this for change detection."""
        return self._version
```

### Key Behaviors

| Method | Thread-safe | Notes |
|--------|-------------|-------|
| `update(ticker, price)` | Yes (Lock) | Creates PriceUpdate with auto-computed `previous_price`; increments version |
| `get(ticker)` | Yes (Lock) | Returns `PriceUpdate` or `None` |
| `get_all()` | Yes (Lock) | Returns shallow copy `dict[str, PriceUpdate]`; safe to iterate outside lock |
| `get_price(ticker)` | Yes (Lock) | Convenience: returns just the `float` price |
| `remove(ticker)` | Yes (Lock) | Used when ticker removed from watchlist |
| `version` | Yes (atomic read) | Monotonically increasing; SSE compares last-sent version to current |

### Version-Based Change Detection

The SSE streaming endpoint uses the version counter to avoid pushing stale data:

```python
last_version = -1
while True:
    current_version = price_cache.version
    if current_version != last_version:
        last_version = current_version
        prices = price_cache.get_all()
        # ... serialize and yield SSE event ...
    await asyncio.sleep(0.5)
```

This is critical for the Massive API source, which only updates every 15 seconds. Without version checking, the SSE stream would re-push identical prices every 500ms, causing false flash animations on the frontend.

---

## 5. Abstract Interface — `interface.py`

The `MarketDataSource` ABC defines the contract that both implementations must follow.

```python
class MarketDataSource(ABC):
    """Contract for market data providers.

    Lifecycle:
        source = create_market_data_source(cache)
        await source.start(["AAPL", "GOOGL", ...])
        # ... app runs ...
        await source.add_ticker("TSLA")
        await source.remove_ticker("GOOGL")
        # ... shutting down ...
        await source.stop()
    """

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Begin producing price updates for the given tickers."""

    @abstractmethod
    async def stop(self) -> None:
        """Stop the background task and release resources."""

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Add a ticker to the active set. No-op if already present."""

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker from the active set. Also removes from PriceCache."""

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Return the current list of actively tracked tickers."""
```

### Contract Guarantees

- **`start()`** must be called exactly once. Creates and starts the background task.
- **`stop()`** is safe to call multiple times. Cancels the background task gracefully.
- **`add_ticker()`** is a no-op if the ticker is already tracked. Takes effect on the next update cycle.
- **`remove_ticker()`** also removes the ticker from the `PriceCache` so it stops appearing in SSE events.
- **`get_tickers()`** is synchronous — returns a copy of the current ticker list.

---

## 6. Seed Prices & Ticker Parameters — `seed_prices.py`

Contains all configuration for the simulator's starting state and behavior.

### Seed Prices

```python
SEED_PRICES: dict[str, float] = {
    "AAPL": 190.00,  "GOOGL": 175.00, "MSFT": 420.00,
    "AMZN": 185.00,  "TSLA": 250.00,  "NVDA": 800.00,
    "META": 500.00,  "JPM": 195.00,   "V": 280.00,
    "NFLX": 600.00,
}
```

### Per-Ticker GBM Parameters

```python
TICKER_PARAMS: dict[str, dict[str, float]] = {
    "AAPL":  {"sigma": 0.22, "mu": 0.05},   # Moderate volatility
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT":  {"sigma": 0.20, "mu": 0.05},
    "AMZN":  {"sigma": 0.28, "mu": 0.05},
    "TSLA":  {"sigma": 0.50, "mu": 0.03},   # High volatility, low drift
    "NVDA":  {"sigma": 0.40, "mu": 0.08},   # High volatility, strong drift
    "META":  {"sigma": 0.30, "mu": 0.05},
    "JPM":   {"sigma": 0.18, "mu": 0.04},   # Low volatility (bank)
    "V":     {"sigma": 0.17, "mu": 0.04},   # Low volatility (payments)
    "NFLX":  {"sigma": 0.35, "mu": 0.05},
}

# For dynamically added tickers not in the list above
DEFAULT_PARAMS: dict[str, float] = {"sigma": 0.25, "mu": 0.05}
```

- **sigma**: Annualized volatility. Higher = larger price swings per tick.
- **mu**: Annualized drift (expected return). Higher = upward bias over time.

### Correlation Groups

```python
CORRELATION_GROUPS: dict[str, set[str]] = {
    "tech":    {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}

INTRA_TECH_CORR    = 0.6   # Tech stocks move together
INTRA_FINANCE_CORR = 0.5   # Finance stocks move together
CROSS_GROUP_CORR   = 0.3   # Between sectors / unknown tickers
TSLA_CORR          = 0.3   # TSLA does its own thing
```

These values are used to build the correlation matrix for the Cholesky decomposition in the simulator (see Section 7).

---

## 7. GBM Simulator — `simulator.py`

The simulator consists of two classes: `GBMSimulator` (pure math engine) and `SimulatorDataSource` (async wrapper implementing `MarketDataSource`).

### 7.1 GBM Mathematics

**Geometric Brownian Motion** models stock prices as:

```
S(t+dt) = S(t) × exp((μ - σ²/2) × dt + σ × √dt × Z)
```

Where:
- `S(t)` = current price
- `μ` (mu) = annualized drift
- `σ` (sigma) = annualized volatility
- `dt` = time step as fraction of a trading year
- `Z` = correlated standard normal random variable

The time step `dt` for 500ms ticks:

```python
TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # = 5,896,800
DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR   # ≈ 8.48e-8
```

This tiny `dt` produces sub-cent moves per tick that accumulate naturally. Over a full simulated trading day (~23,400 ticks), the cumulative drift and volatility match realistic annualized values.

### 7.2 Correlated Moves via Cholesky Decomposition

Stocks don't move independently. The simulator uses a **Cholesky decomposition** of the correlation matrix to generate correlated random draws.

```
1. Generate n independent standard normal draws: Z_independent
2. Build correlation matrix C from sector groupings
3. Compute Cholesky factor L where C = L × L^T
4. Correlated draws: Z_correlated = L × Z_independent
```

The correlation matrix is rebuilt whenever tickers are added or removed. With n < 50 tickers, this O(n²) operation is negligible.

**Pairwise correlation logic:**

```python
@staticmethod
def _pairwise_correlation(t1: str, t2: str) -> float:
    tech = CORRELATION_GROUPS["tech"]
    finance = CORRELATION_GROUPS["finance"]

    if t1 == "TSLA" or t2 == "TSLA":
        return TSLA_CORR         # 0.3 — TSLA does its own thing
    if t1 in tech and t2 in tech:
        return INTRA_TECH_CORR   # 0.6
    if t1 in finance and t2 in finance:
        return INTRA_FINANCE_CORR  # 0.5
    return CROSS_GROUP_CORR       # 0.3
```

### 7.3 Random Shock Events

For visual drama, the simulator injects random "event" moves:

```python
# ~0.1% chance per tick per ticker
# With 10 tickers at 2 ticks/sec → expect an event ~every 50 seconds
if random.random() < self._event_prob:       # default: 0.001
    shock_magnitude = random.uniform(0.02, 0.05)  # 2-5%
    shock_sign = random.choice([-1, 1])
    self._prices[ticker] *= 1 + shock_magnitude * shock_sign
```

### 7.4 GBMSimulator Core Step

The hot path — called every 500ms:

```python
def step(self) -> dict[str, float]:
    n = len(self._tickers)
    if n == 0:
        return {}

    z_independent = np.random.standard_normal(n)
    z_correlated = self._cholesky @ z_independent if self._cholesky is not None else z_independent

    result: dict[str, float] = {}
    for i, ticker in enumerate(self._tickers):
        params = self._params[ticker]
        mu, sigma = params["mu"], params["sigma"]

        drift = (mu - 0.5 * sigma**2) * self._dt
        diffusion = sigma * math.sqrt(self._dt) * z_correlated[i]
        self._prices[ticker] *= math.exp(drift + diffusion)

        # Random shock events
        if random.random() < self._event_prob:
            shock = random.uniform(0.02, 0.05) * random.choice([-1, 1])
            self._prices[ticker] *= 1 + shock

        result[ticker] = round(self._prices[ticker], 2)

    return result
```

### 7.5 Dynamic Ticker Management

```python
def add_ticker(self, ticker: str) -> None:
    """Add a ticker. Seeds price from SEED_PRICES or random [50, 300]."""
    if ticker in self._prices:
        return
    self._add_ticker_internal(ticker)
    self._rebuild_cholesky()  # Rebuild correlation matrix

def remove_ticker(self, ticker: str) -> None:
    if ticker not in self._prices:
        return
    self._tickers.remove(ticker)
    del self._prices[ticker]
    del self._params[ticker]
    self._rebuild_cholesky()

def _add_ticker_internal(self, ticker: str) -> None:
    """Add without rebuilding Cholesky (used during batch init)."""
    self._tickers.append(ticker)
    self._prices[ticker] = SEED_PRICES.get(ticker, random.uniform(50.0, 300.0))
    self._params[ticker] = TICKER_PARAMS.get(ticker, dict(DEFAULT_PARAMS))
```

### 7.6 SimulatorDataSource (Async Wrapper)

Wraps `GBMSimulator` as a `MarketDataSource`, running the step loop as an asyncio background task:

```python
class SimulatorDataSource(MarketDataSource):
    def __init__(self, price_cache: PriceCache, update_interval: float = 0.5,
                 event_probability: float = 0.001) -> None:
        self._cache = price_cache
        self._interval = update_interval
        self._event_prob = event_probability
        self._sim: GBMSimulator | None = None
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(tickers=tickers, event_probability=self._event_prob)
        # Seed cache with initial prices so SSE has data immediately
        for ticker in tickers:
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)
        self._task = asyncio.create_task(self._run_loop(), name="simulator-loop")

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None

    async def add_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.add_ticker(ticker)
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)

    async def remove_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.remove_ticker(ticker)
        self._cache.remove(ticker)

    def get_tickers(self) -> list[str]:
        return self._sim.get_tickers() if self._sim else []

    async def _run_loop(self) -> None:
        while True:
            try:
                if self._sim:
                    prices = self._sim.step()
                    for ticker, price in prices.items():
                        self._cache.update(ticker=ticker, price=price)
            except Exception:
                logger.exception("Simulator step failed")
            await asyncio.sleep(self._interval)
```

### Timing Characteristics

| Parameter | Value | Effect |
|---|---|---|
| `update_interval` | 0.5s | Two price updates per second per ticker |
| `event_probability` | 0.001 | ~0.1% chance of 2-5% shock per tick per ticker |
| Expected event frequency | ~1 per 50s | With 10 tickers × 2 ticks/s = 20 ticks/s |

---

## 8. Massive API Client — `massive_client.py`

The Massive (Polygon.io) data source polls the REST API for real market data.

### 8.1 Implementation

```python
class MassiveDataSource(MarketDataSource):
    def __init__(self, api_key: str, price_cache: PriceCache,
                 poll_interval: float = 15.0) -> None:
        self._api_key = api_key
        self._cache = price_cache
        self._interval = poll_interval
        self._tickers: list[str] = []
        self._task: asyncio.Task | None = None
        self._client: RESTClient | None = None

    async def start(self, tickers: list[str]) -> None:
        self._client = RESTClient(api_key=self._api_key)
        self._tickers = list(tickers)
        await self._poll_once()  # Immediate first poll
        self._task = asyncio.create_task(self._poll_loop(), name="massive-poller")

    async def _poll_once(self) -> None:
        if not self._tickers or not self._client:
            return
        try:
            # RESTClient is synchronous — run in thread to avoid blocking event loop
            snapshots = await asyncio.to_thread(self._fetch_snapshots)
            for snap in snapshots:
                try:
                    price = snap.last_trade.price
                    timestamp = snap.last_trade.timestamp / 1000.0  # ms → seconds
                    self._cache.update(ticker=snap.ticker, price=price, timestamp=timestamp)
                except (AttributeError, TypeError) as e:
                    logger.warning("Skipping snapshot for %s: %s",
                                   getattr(snap, "ticker", "???"), e)
        except Exception as e:
            logger.error("Massive poll failed: %s", e)

    def _fetch_snapshots(self) -> list:
        return self._client.get_snapshot_all(
            market_type=SnapshotMarketType.STOCKS,
            tickers=self._tickers,
        )
```

### 8.2 Key Design Details

**Single API call for all tickers**: The `get_snapshot_all()` endpoint accepts a list of tickers and returns all snapshots in one response. This is essential for staying within the free tier's 5 requests/minute limit.

**Thread offloading**: The `massive` Python SDK is synchronous. `asyncio.to_thread()` runs the blocking HTTP call in a thread pool worker so it doesn't block the event loop.

**Ticker normalization**: `add_ticker()` and `remove_ticker()` normalize to uppercase via `.upper().strip()`. The simulator does not do this (noted as a known issue in the review — see Section 14).

**Resilient polling**: If a poll fails (401, 429, network error), it logs the error and continues. The next poll cycle will retry automatically.

### 8.3 Rate Limit Considerations

| Tier | Limit | Recommended `poll_interval` |
|------|-------|-----------------------------|
| Free | 5 req/min | 15.0s (default) |
| Starter | Unlimited | 5.0s |
| Developer+ | Unlimited | 2.0s |

The poll interval is configurable via the constructor but defaults to 15s to be safe for free-tier users.

### 8.4 SSE Behavior with Massive Source

Since Massive only polls every 15 seconds, the SSE stream's version-based change detection prevents re-pushing stale data. The stream checks `price_cache.version` every 500ms — if no poll has occurred since the last push, the version is unchanged and no event is sent. This means:

- Clients see price updates in bursts every ~15 seconds (not the smooth 500ms cadence of the simulator)
- No false flash animations on the frontend from stale re-pushes
- The `retry: 1000` SSE directive ensures auto-reconnection if the connection drops

---

## 9. Factory — `factory.py`

A simple factory function selects the data source based on the `MASSIVE_API_KEY` environment variable.

```python
def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    """Create the appropriate market data source based on environment variables.

    - MASSIVE_API_KEY set and non-empty → MassiveDataSource (real market data)
    - Otherwise → SimulatorDataSource (GBM simulation)
    """
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()

    if api_key:
        logger.info("Market data source: Massive API (real data)")
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    else:
        logger.info("Market data source: GBM Simulator")
        return SimulatorDataSource(price_cache=price_cache)
```

**Returns an unstarted source.** The caller must `await source.start(tickers)` to begin producing prices.

---

## 10. SSE Streaming Endpoint — `stream.py`

### 10.1 Router Factory

```python
router = APIRouter(prefix="/api/stream", tags=["streaming"])

def create_stream_router(price_cache: PriceCache) -> APIRouter:
    @router.get("/prices")
    async def stream_prices(request: Request) -> StreamingResponse:
        return StreamingResponse(
            _generate_events(price_cache, request),
            media_type="text/event-stream",
            headers={
                "Cache-Control": "no-cache",
                "Connection": "keep-alive",
                "X-Accel-Buffering": "no",
            },
        )
    return router
```

**Endpoint**: `GET /api/stream/prices`
**Content-Type**: `text/event-stream`

### 10.2 Event Generator

```python
async def _generate_events(
    price_cache: PriceCache,
    request: Request,
    interval: float = 0.5,
) -> AsyncGenerator[str, None]:
    yield "retry: 1000\n\n"  # Auto-reconnect after 1s

    last_version = -1
    try:
        while True:
            if await request.is_disconnected():
                break

            current_version = price_cache.version
            if current_version != last_version:
                last_version = current_version
                prices = price_cache.get_all()
                if prices:
                    data = {ticker: update.to_dict() for ticker, update in prices.items()}
                    payload = json.dumps(data)
                    yield f"data: {payload}\n\n"

            await asyncio.sleep(interval)
    except asyncio.CancelledError:
        pass
```

### 10.3 SSE Event Format

Each event contains all tracked tickers in a single JSON object:

```
retry: 1000

data: {"AAPL": {"ticker": "AAPL", "price": 191.23, "previous_price": 190.87, "timestamp": 1710172800.5, "change": 0.36, "change_percent": 0.1886, "direction": "up"}, "GOOGL": {"ticker": "GOOGL", "price": 176.10, ...}, ...}

data: {"AAPL": {"ticker": "AAPL", "price": 191.45, ...}, ...}
```

### 10.4 Frontend Integration

```javascript
const eventSource = new EventSource('/api/stream/prices');

eventSource.onmessage = (event) => {
    const prices = JSON.parse(event.data);
    // prices = { "AAPL": { ticker, price, previous_price, change, direction, ... }, ... }
    for (const [ticker, update] of Object.entries(prices)) {
        updateWatchlist(ticker, update);
        if (update.direction === 'up') flashGreen(ticker);
        if (update.direction === 'down') flashRed(ticker);
    }
};

eventSource.onerror = () => {
    // EventSource auto-reconnects using the retry: 1000 directive
    updateConnectionStatus('reconnecting');
};
```

---

## 11. FastAPI Lifecycle Integration

The market data source must be started during app startup and stopped during shutdown. This is done via FastAPI's `lifespan` async context manager (not the deprecated `@app.on_event`).

### Recommended Integration in `main.py`

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.market import PriceCache, create_market_data_source, create_stream_router

@asynccontextmanager
async def lifespan(app: FastAPI):
    # --- Startup ---
    cache = PriceCache()
    source = create_market_data_source(cache)

    # Load initial tickers from the database watchlist
    tickers = get_watchlist_tickers()  # e.g., ["AAPL", "GOOGL", ...]
    await source.start(tickers)

    # Store references for request handlers
    app.state.price_cache = cache
    app.state.market_source = source

    yield

    # --- Shutdown ---
    await source.stop()

app = FastAPI(lifespan=lifespan)

# Mount SSE streaming router
stream_router = create_stream_router(app.state.price_cache)
app.include_router(stream_router)
```

### Access from Request Handlers

```python
@app.post("/api/watchlist")
async def add_to_watchlist(request: Request, body: WatchlistAdd):
    source: MarketDataSource = request.app.state.market_source
    cache: PriceCache = request.app.state.price_cache

    # Add to database
    db_add_watchlist(body.ticker)

    # Tell the data source to start tracking this ticker
    await source.add_ticker(body.ticker)

    # Return current price (if available immediately)
    price = cache.get(body.ticker)
    return {"ticker": body.ticker, "price": price.to_dict() if price else None}
```

---

## 12. Watchlist Coordination

When tickers are added or removed from the watchlist (via API or LLM chat), the market data source and price cache must stay in sync.

### Add Ticker Flow

```
User adds "PYPL" to watchlist
    │
    ├── 1. Insert into SQLite `watchlist` table
    │
    ├── 2. await source.add_ticker("PYPL")
    │       ├── Simulator: adds to GBMSimulator, seeds price, rebuilds Cholesky
    │       └── Massive: appends to ticker list (appears on next poll)
    │
    └── 3. Cache now includes "PYPL"
            └── SSE stream picks it up on next push cycle
```

### Remove Ticker Flow

```
User removes "PYPL" from watchlist
    │
    ├── 1. Delete from SQLite `watchlist` table
    │
    ├── 2. await source.remove_ticker("PYPL")
    │       ├── Simulator: removes from GBMSimulator, rebuilds Cholesky
    │       ├── Massive: removes from ticker list
    │       └── Both: calls cache.remove("PYPL")
    │
    └── 3. Cache no longer includes "PYPL"
            └── SSE stream stops sending it
```

### Timing Differences Between Sources

| Action | Simulator | Massive |
|--------|-----------|---------|
| Add ticker → first price in cache | Immediate (seeds from `SEED_PRICES`) | Next poll cycle (up to 15s) |
| Remove ticker → removed from cache | Immediate | Immediate |
| Add ticker → appears in SSE | Next 500ms cycle | Next poll + next SSE cycle |

---

## 13. Data Flow Diagrams

### Simulator Data Flow (Default)

```
Every 500ms:
┌─────────────┐     step()      ┌─────────────┐    update()     ┌─────────────┐
│ GBMSimulator │ ──────────────→ │Simulator     │ ──────────────→│ PriceCache  │
│              │  {ticker: price}│DataSource    │  per ticker    │             │
│ • GBM math   │                 │ • async loop │                │ • lock      │
│ • Cholesky   │                 │ • exception  │                │ • version++ │
│ • shocks     │                 │   handling   │                │             │
└─────────────┘                  └─────────────┘                 └──────┬──────┘
                                                                        │
                                                                  reads │
                                                                        ▼
                                                                 ┌─────────────┐
                                                                 │ SSE stream  │
                                                                 │ (500ms poll)│
                                                                 │             │
                                                                 │→ JSON event │
                                                                 └─────────────┘
```

### Massive Data Flow (Optional)

```
Every 15s:
┌─────────────┐  get_snapshot_all()  ┌─────────────┐   update()    ┌─────────────┐
│ Polygon.io  │ ←───────────────────│Massive       │──────────────→│ PriceCache  │
│ REST API    │ ───────────────────→│DataSource    │  per ticker   │             │
│             │  JSON snapshots     │ • to_thread  │               │ • version++ │
│             │                     │ • error log  │               │ only when   │
│             │                     │ • retry next │               │ price ≠ prev│
└─────────────┘                     └─────────────┘                └──────┬──────┘
                                                                          │
                                                                    reads │
                                                                          ▼
                                                                   ┌─────────────┐
                                                                   │ SSE stream  │
                                                                   │ (500ms poll)│
                                                                   │ version chk │
                                                                   │→ only when  │
                                                                   │  changed    │
                                                                   └─────────────┘
```

---

## 14. Error Handling & Edge Cases

### Simulator Errors

| Scenario | Handling |
|---|---|
| `step()` raises exception | Caught in `_run_loop`; logged; loop continues next cycle |
| Cholesky decomposition fails (non-positive-definite matrix) | Would only happen with correlation > 1.0 or < -1.0; prevented by constant values |
| Empty ticker list | `step()` returns `{}`; no-op |

### Massive API Errors

| Scenario | Handling |
|---|---|
| 401 Invalid API key | Logged as error; poll loop continues (all polls will fail until key is fixed) |
| 429 Rate limit exceeded | Logged; next poll retries after the interval |
| Network timeout/error | Logged; next poll retries |
| Ticker not found in snapshot response | Silently skipped (that ticker just won't appear in cache) |
| Malformed snapshot (missing `last_trade`) | `AttributeError` caught per-snapshot; other tickers still processed |

### SSE Stream Errors

| Scenario | Handling |
|---|---|
| Client disconnects | `request.is_disconnected()` returns True; generator exits cleanly |
| Client reconnects | `EventSource` auto-reconnects per `retry: 1000` directive; new generator starts |
| `CancelledError` on server shutdown | Caught in generator; exits cleanly |

### Known Issues (from Code Review)

1. **Inconsistent ticker case normalization**: `MassiveDataSource` normalizes to uppercase; `SimulatorDataSource` does not. If a ticker is added in lowercase, the simulator creates a separate key. Both should normalize to uppercase. (See `planning/REVIEW.md` §3.1)

2. **Module-level router in `stream.py`**: `create_stream_router()` attaches routes to a module-level `router` object. If called more than once (e.g., in tests), duplicate routes accumulate. The router should be created inside the factory function. (See `planning/REVIEW.md` §3.2)

---

## 15. Testing Strategy

73 tests across 6 modules, all passing. Overall coverage: 84%.

### Test Modules

| Module | Tests | What It Covers |
|--------|-------|----------------|
| `test_models.py` | 11 | PriceUpdate construction, properties (change, change_percent, direction), to_dict serialization, edge cases (zero previous_price) |
| `test_cache.py` | 13 | PriceCache CRUD, thread safety, version counter, remove behavior, get_all returns copy |
| `test_simulator.py` | 17 | GBM math correctness, Cholesky correlation, random events, add/remove tickers, step() output shape |
| `test_simulator_source.py` | 10 | SimulatorDataSource lifecycle (start/stop), cache integration, add_ticker seeds cache, remove_ticker clears cache |
| `test_factory.py` | 7 | Environment variable selection logic, returns correct type, unstarted state |
| `test_massive.py` | 13 | Snapshot parsing, error handling (malformed data, API errors), ticker normalization, poll loop behavior |

### Running Tests

```bash
cd backend
uv run --extra dev pytest -v                      # All tests, verbose
uv run --extra dev pytest tests/market/ -v         # Market tests only
uv run --extra dev pytest --cov=app --cov-report=term-missing  # With coverage
```

### Testing Patterns Used

**PriceCache tests** — Direct unit tests, no mocking needed:
```python
def test_update_creates_price_update():
    cache = PriceCache()
    update = cache.update("AAPL", 190.50)
    assert update.ticker == "AAPL"
    assert update.price == 190.50
    assert update.previous_price == 190.50  # First update: prev == current
    assert update.direction == "flat"

def test_version_increments():
    cache = PriceCache()
    assert cache.version == 0
    cache.update("AAPL", 190.0)
    assert cache.version == 1
    cache.update("AAPL", 191.0)
    assert cache.version == 2
```

**Simulator tests** — Test the math engine directly:
```python
def test_step_returns_all_tickers():
    sim = GBMSimulator(["AAPL", "GOOGL", "MSFT"])
    prices = sim.step()
    assert set(prices.keys()) == {"AAPL", "GOOGL", "MSFT"}
    assert all(isinstance(p, float) for p in prices.values())
    assert all(p > 0 for p in prices.values())

def test_add_ticker_includes_in_step():
    sim = GBMSimulator(["AAPL"])
    sim.add_ticker("TSLA")
    prices = sim.step()
    assert "TSLA" in prices
```

**Factory tests** — Patch environment variables:
```python
def test_creates_simulator_by_default(monkeypatch):
    monkeypatch.delenv("MASSIVE_API_KEY", raising=False)
    cache = PriceCache()
    source = create_market_data_source(cache)
    assert isinstance(source, SimulatorDataSource)

def test_creates_massive_with_api_key(monkeypatch):
    monkeypatch.setenv("MASSIVE_API_KEY", "test-key-123")
    cache = PriceCache()
    source = create_market_data_source(cache)
    assert isinstance(source, MassiveDataSource)
```

**Massive tests** — Mock the REST client:
```python
@pytest.fixture
def massive_source():
    cache = PriceCache()
    source = MassiveDataSource(api_key="test-key", price_cache=cache)
    source._client = MagicMock()  # Mock the REST client
    return source, cache
```

---

## 16. Configuration Summary

| Parameter | Location | Default | Description |
|---|---|---|---|
| `MASSIVE_API_KEY` | Environment variable | Empty (simulator) | If set, uses Massive API for real data |
| Simulator update interval | `SimulatorDataSource.__init__` | 0.5s | Time between GBM steps |
| Simulator event probability | `SimulatorDataSource.__init__` | 0.001 | Per-tick-per-ticker shock chance |
| Massive poll interval | `MassiveDataSource.__init__` | 15.0s | Time between API polls |
| SSE push interval | `_generate_events()` | 0.5s | Time between SSE version checks |
| SSE retry directive | `_generate_events()` | 1000ms | Client auto-reconnect delay |
| GBM dt | `GBMSimulator.DEFAULT_DT` | ~8.48e-8 | Time step as fraction of trading year |
| Seed prices | `seed_prices.py` | 10 tickers | Starting prices for default watchlist |
| Ticker params (sigma, mu) | `seed_prices.py` | Per-ticker | Annualized volatility and drift |
| Default params | `seed_prices.py` | σ=0.25, μ=0.05 | For dynamically added tickers |
| Correlation (tech intra) | `seed_prices.py` | 0.6 | Tech stocks move together |
| Correlation (finance intra) | `seed_prices.py` | 0.5 | Finance stocks move together |
| Correlation (cross-sector) | `seed_prices.py` | 0.3 | Between sectors |
| Correlation (TSLA) | `seed_prices.py` | 0.3 | TSLA with anything |

---

## Downstream Consumer Quick Reference

For agents building portfolio, trade, watchlist, or chat features — here's how to interact with the market data subsystem:

### Get Current Price for Trade Execution

```python
cache: PriceCache = request.app.state.price_cache
price = cache.get_price("AAPL")  # float or None
if price is None:
    raise HTTPException(400, f"No price available for AAPL")
```

### Get All Prices for Portfolio Valuation

```python
all_prices = cache.get_all()  # dict[str, PriceUpdate]
for ticker, update in all_prices.items():
    current_value = position_qty * update.price
```

### Add/Remove Tickers from Watchlist

```python
source: MarketDataSource = request.app.state.market_source
await source.add_ticker("PYPL")     # Starts tracking
await source.remove_ticker("PYPL")  # Stops tracking + removes from cache
```

### Get Tracked Tickers

```python
tickers = source.get_tickers()  # ["AAPL", "GOOGL", ...]
```
