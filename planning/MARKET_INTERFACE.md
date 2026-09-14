# Market Data Interface Design

Unified Python interface for market data in FinAlly. Two implementations — the GBM simulator and the Massive API client — sit behind one abstract interface and write into one shared cache, so everything downstream (SSE streaming, portfolio valuation, trade execution) is source-agnostic.

This document describes the design **as implemented** in `backend/app/market/` (verified against the current source on 2026-09-14) and supersedes `planning/archive/MARKET_INTERFACE.md`, whose code sketches drifted slightly from what shipped — see [Differences from the earlier sketch](#differences-from-the-earlier-sketch). For the live-data side of this, see [MASSIVE_API.md](MASSIVE_API.md); for the simulator internals, see [MARKET_SIMULATOR.md](MARKET_SIMULATOR.md).

## Core Data Model — `models.py`

```python
from dataclasses import dataclass, field
import time

@dataclass(frozen=True, slots=True)
class PriceUpdate:
    """Immutable snapshot of a single ticker's price at a point in time."""
    ticker: str
    price: float
    previous_price: float
    timestamp: float = field(default_factory=time.time)  # Unix seconds

    @property
    def change(self) -> float: ...          # round(price - previous_price, 4)

    @property
    def change_percent(self) -> float: ...  # round(pct change, 4); 0.0 if previous_price == 0

    @property
    def direction(self) -> str: ...          # "up" / "down" / "flat"

    def to_dict(self) -> dict: ...           # JSON-serializable form for SSE
```

`PriceUpdate` is the only object that leaves the market data layer. `change`, `change_percent`, and `direction` are **computed properties**, not stored fields — the cache only needs to store `price`/`previous_price`/`timestamp`, and every reader derives the rest consistently. This is the one deliberate simplification versus the original sketch (which stored `change`/`direction` as plain fields computed once at cache-write time); computing them as properties means a `PriceUpdate` is self-consistent even if constructed directly (e.g. in tests) without going through `PriceCache`.

## Abstract Interface — `interface.py`

```python
from abc import ABC, abstractmethod

class MarketDataSource(ABC):
    """Contract for market data providers.

    Implementations push price updates into a shared PriceCache on their own
    schedule. Downstream code never calls the data source directly for prices
    -- it reads from the cache.
    """

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Begin producing updates. Starts a background task. Call once."""

    @abstractmethod
    async def stop(self) -> None:
        """Stop the background task and release resources. Safe to call twice."""

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Add a ticker to the active set. No-op if already present."""

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker. Also removes it from the PriceCache."""

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Currently tracked tickers."""
```

Both `SimulatorDataSource` and `MassiveDataSource` implement this ABC and nothing else — the interface has no method that returns a price directly. That's intentional: prices flow one way, from source → cache → readers, so a caller can never accidentally read a stale value straight off the source object.

## Price Cache — `cache.py`

The single point of truth that both sources write to and every reader (SSE stream, portfolio valuation, trade execution) reads from.

```python
class PriceCache:
    """Thread-safe in-memory cache of the latest price for each ticker."""

    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
        """Record a new price. Computes previous_price from what was cached before
        (or price itself, on first sight -> direction='flat'). Rounds price to 2dp.
        Bumps the version counter."""

    def get(self, ticker: str) -> PriceUpdate | None: ...
    def get_price(self, ticker: str) -> float | None: ...   # convenience: just the float
    def get_all(self) -> dict[str, PriceUpdate]: ...          # shallow-copy snapshot
    def remove(self, ticker: str) -> None: ...

    @property
    def version(self) -> int: ...   # monotonic counter, bumped on every update()
```

Two details worth calling out because they matter to correctness, not just style:

- **`previous_price` comes from the cache's own last value, never from the data source.** Neither `SimulatorDataSource` nor `MassiveDataSource` computes or passes `previous_price` — they only ever call `cache.update(ticker, price, timestamp)` with the new price. This means `direction`/`change` are always "change since the last tick FinAlly itself observed," which behaves identically whether the source is the simulator or Massive, and avoids trusting Massive's own `todaysChange`/`prevDay.close` fields (which reset at Massive's daily 3:30 AM ET data-clear boundary, not at a boundary meaningful to this app — see [MASSIVE_API.md](MASSIVE_API.md)).
- **`version` is a cheap change-detection signal for the SSE loop**, so the stream doesn't re-serialize and push an unchanged snapshot every tick — see `stream.py` below. It's a plain `int`, guarded by the same lock as the price dict, incremented once per `update()` call (so N ticker updates in one poll cycle bump it N times, which is fine — the SSE loop only compares "did it change since I last looked").

## Factory — `factory.py`

```python
def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    """MASSIVE_API_KEY set and non-empty -> MassiveDataSource.
    Otherwise -> SimulatorDataSource (GBM).
    Returns an unstarted source; caller must await source.start(tickers)."""
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()
    if api_key:
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    return SimulatorDataSource(price_cache=price_cache)
```

Trivial by design — this is the single decision point for the whole app, and it's a pure function of one environment variable. No retries, no fallback-on-failure logic here: if `MASSIVE_API_KEY` is set but the key turns out to be invalid or the wrong plan tier, the caller finds out at `start()`/first poll, not here (see caveat below).

**Design note — a known gap, not yet implemented:** as documented in [MASSIVE_API.md](MASSIVE_API.md), a free/Basic-tier `MASSIVE_API_KEY` cannot call the snapshot endpoints `MassiveDataSource` depends on — every poll will 403. Today that surfaces only as a logged error every `poll_interval` seconds (`MassiveDataSource._poll_once` catches and logs, it doesn't raise) with a cache that never gets populated for that source. If this project wants to be forgiving of a free-tier key, the factory or the poller would need to either (a) detect the 403 and fall back to `SimulatorDataSource`, or (b) fall back to polling `get_grouped_daily_aggs`/`get_previous_close_agg` (both free-tier-accessible) instead of `get_snapshot_all`, at EOD-refresh cadence rather than intraday. Neither is implemented; flagging it here so it isn't rediscovered as a "silent failure" bug later.

## Massive Implementation — `massive_client.py`

```python
class MassiveDataSource(MarketDataSource):
    """Polls GET /v2/snapshot/locale/us/markets/stocks/tickers for all watched
    tickers in a single API call, writes results to the PriceCache.

    Rate limits: free tier 5 req/min -> poll every 15s (default).
                 paid tiers -> poll every 2-5s.
    """

    def __init__(self, api_key: str, price_cache: PriceCache, poll_interval: float = 15.0): ...

    async def start(self, tickers: list[str]) -> None:
        # Creates the RESTClient, does one immediate poll (so the cache isn't
        # empty for the first 15s), then starts the background poll loop.
        ...

    async def _poll_once(self) -> None:
        # Runs the synchronous RESTClient call in a thread (asyncio.to_thread)
        # to avoid blocking the event loop. Extracts snap.last_trade.price and
        # snap.last_trade.timestamp (ns -> s). Catches all exceptions per cycle
        # and logs -- a failed poll never crashes the loop or raises to the caller.
```

The synchronous-client-in-a-thread pattern (`asyncio.to_thread`) is required because `massive.RESTClient` is a blocking `requests`-based client with no async variant — running it directly on the event loop would stall SSE delivery to every connected browser for the duration of each HTTP round-trip.

## Simulator Implementation — `simulator.py`

```python
class SimulatorDataSource(MarketDataSource):
    """Runs a background asyncio task that calls GBMSimulator.step() every
    update_interval seconds (default 500ms) and writes results to the cache."""

    def __init__(self, price_cache: PriceCache, update_interval: float = 0.5,
                 event_probability: float = 0.001): ...

    async def start(self, tickers: list[str]) -> None:
        # Builds a GBMSimulator, immediately seeds the cache with starting
        # prices (so SSE has data before the first 500ms tick), then starts
        # the loop.
```

Full simulator math and structure: [MARKET_SIMULATOR.md](MARKET_SIMULATOR.md).

## SSE Integration — `stream.py`

```python
def create_stream_router(price_cache: PriceCache) -> APIRouter:
    """Factory (not a global router) so the PriceCache can be injected
    without module-level state. Endpoint: GET /api/stream/prices."""
```

The generator loop polls `price_cache.version` every 500ms; only when the version has changed since the last send does it call `get_all()` and serialize+yield. It also checks `request.is_disconnected()` each tick to stop the generator promptly when the browser navigates away, and sends `retry: 1000\n\n` up front so `EventSource`'s built-in auto-reconnect retries after 1s rather than the browser's default (which varies).

## File Structure (as shipped)

```
backend/
  app/
    market/
      __init__.py
      models.py             # PriceUpdate
      interface.py          # MarketDataSource ABC
      cache.py              # PriceCache
      factory.py            # create_market_data_source()
      massive_client.py     # MassiveDataSource
      simulator.py           # GBMSimulator + SimulatorDataSource
      seed_prices.py         # SEED_PRICES, TICKER_PARAMS, correlation constants
      stream.py              # create_stream_router() — SSE endpoint factory
  tests/
    market/                  # 73 tests across 6 modules, ~84% coverage
```

## Lifecycle

1. **App startup**: `cache = PriceCache()`; `source = create_market_data_source(cache)`; `await source.start(initial_tickers)`.
2. **Watchlist changes**: `await source.add_ticker(t)` / `await source.remove_ticker(t)`.
3. **SSE streaming**: `create_stream_router(cache)`, mounted once; reads `cache.version` / `cache.get_all()` every 500ms per connected client.
4. **Trade execution / portfolio valuation**: `cache.get_price(ticker)`.
5. **App shutdown**: `await source.stop()`.

## Differences from the earlier sketch

`planning/archive/MARKET_INTERFACE.md` was written before implementation and got the shape right, but three things changed during build and are worth recording so future readers trust this doc over that one:

1. **`change`/`direction` moved from stored fields to computed properties** on `PriceUpdate` (see [Core Data Model](#core-data-model--modelspy) above) — a robustness improvement, not a functional change.
2. **`PriceCache` gained a `version` counter and a `get_price()` convenience method**, neither of which was in the original sketch. `version` is what makes the SSE loop cheap (avoid re-diffing dicts every 500ms); it's a small but load-bearing addition.
3. **The Massive sketch's `poll_once` read `day.previous_close`**, a field that doesn't exist on the real API (see [MASSIVE_API.md](MASSIVE_API.md#corrections-vs-the-archived-version)). The shipped `massive_client.py` never adopted that field — it only ever reads `last_trade.price`/`last_trade.timestamp` and lets `PriceCache` derive `previous_price` itself, which sidestepped the bug before it could ship.
