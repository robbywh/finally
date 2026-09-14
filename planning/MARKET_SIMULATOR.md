# Market Simulator Design

Approach and code structure for simulating realistic stock prices when no `MASSIVE_API_KEY` is configured (the default). This document describes the simulator **as implemented** in `backend/app/market/simulator.py` and `seed_prices.py` (verified against the current source on 2026-09-14) and supersedes `planning/archive/MARKET_SIMULATOR.md`. See [MARKET_INTERFACE.md](MARKET_INTERFACE.md) for how `SimulatorDataSource` plugs into the shared `MarketDataSource`/`PriceCache` architecture.

## Overview

The simulator uses **Geometric Brownian Motion (GBM)** — the standard continuous-time model underlying Black-Scholes — to generate price paths that are always positive, exhibit the lognormal returns distribution seen in real markets, and can be tuned per-ticker for drift and volatility. `SimulatorDataSource` drives it from an asyncio loop at ~500ms intervals, which is fast enough to feel "live" without generating more SSE traffic than a browser needs.

## GBM Math

At each time step, price evolves as:

```
S(t+dt) = S(t) * exp((mu - sigma^2/2) * dt + sigma * sqrt(dt) * Z)
```

Where:
- `S(t)` = current price
- `mu` = annualized drift (expected return), e.g. `0.05` (5%/year)
- `sigma` = annualized volatility, e.g. `0.20` (20%/year)
- `dt` = this time step, expressed as a fraction of a trading year
- `Z` = a standard normal random draw (correlated across tickers — see below)

`dt` for a 500ms tick, given 252 trading days/year and 6.5 trading hours/day:

```
TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600 = 5,896,800
DEFAULT_DT = 0.5 / 5,896,800  ≈ 8.48e-8
```

This tiny `dt` keeps individual ticks to sub-cent moves that accumulate into realistic-looking intraday ranges over minutes, rather than producing a visibly jumpy random walk.

## Correlated Moves — Cholesky Decomposition

Real stocks don't move independently — sector peers tend to move together. Given a correlation matrix `C`, `L = cholesky(C)` gives a lower-triangular matrix such that, for independent standard normals `z_independent`, `z_correlated = L @ z_independent` has covariance structure `C`. This is recomputed whenever the tracked ticker set changes (add/remove), since the matrix dimension depends on `n = len(tickers)`.

Correlation structure (`seed_prices.py`):

| Group | Members | Intra-group correlation |
|---|---|---|
| **Tech** | AAPL, GOOGL, MSFT, AMZN, META, NVDA, NFLX | `INTRA_TECH_CORR = 0.6` |
| **Finance** | JPM, V | `INTRA_FINANCE_CORR = 0.5` |
| **TSLA** | (its own case) | `TSLA_CORR = 0.3` with *everything*, including other tech names |
| Cross-group / unknown tickers | anything else | `CROSS_GROUP_CORR = 0.3` |

`_pairwise_correlation(t1, t2)` checks the TSLA special case **first** — TSLA is nominally in the tech set by ticker but is deliberately carved out to correlate at only 0.3 with its sector peers, "doing its own thing" the way a high-beta, narrative-driven stock does in practice, rather than moving in lockstep with AAPL/MSFT.

## Random Shock Events

Each tick, each ticker independently has a small chance (`event_probability`, default `0.001` = 0.1%) of an extra sudden move layered on top of the GBM step:

```python
if random.random() < event_probability:
    shock = random.uniform(0.02, 0.05) * random.choice([-1, 1])
    price *= (1 + shock)
```

At 2 ticks/sec, 0.1% per tick per ticker works out to roughly one shock event per ticker every ~500 seconds. With the default 10-ticker watchlist, that's a shock somewhere in the watchlist roughly every ~50 seconds — frequent enough to give sparklines and the flash animations something dramatic to show, without every ticker visibly jumping every few seconds.

## Seed Prices & Per-Ticker Parameters — `seed_prices.py`

```python
SEED_PRICES: dict[str, float] = {
    "AAPL": 190.00, "GOOGL": 175.00, "MSFT": 420.00, "AMZN": 185.00,
    "TSLA": 250.00, "NVDA": 800.00, "META": 500.00, "JPM": 195.00,
    "V": 280.00, "NFLX": 600.00,
}

TICKER_PARAMS: dict[str, dict[str, float]] = {
    "AAPL":  {"sigma": 0.22, "mu": 0.05},
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT":  {"sigma": 0.20, "mu": 0.05},
    "AMZN":  {"sigma": 0.28, "mu": 0.05},
    "TSLA":  {"sigma": 0.50, "mu": 0.03},   # high vol, modest drift
    "NVDA":  {"sigma": 0.40, "mu": 0.08},   # high vol, strong drift
    "META":  {"sigma": 0.30, "mu": 0.05},
    "JPM":   {"sigma": 0.18, "mu": 0.04},   # low vol (bank)
    "V":     {"sigma": 0.17, "mu": 0.04},   # low vol (payments)
    "NFLX":  {"sigma": 0.35, "mu": 0.05},
}

DEFAULT_PARAMS: dict[str, float] = {"sigma": 0.25, "mu": 0.05}  # unknown/dynamically-added tickers

CORRELATION_GROUPS: dict[str, set[str]] = {
    "tech": {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}
INTRA_TECH_CORR = 0.6
INTRA_FINANCE_CORR = 0.5
CROSS_GROUP_CORR = 0.3
TSLA_CORR = 0.3
```

A ticker added to the watchlist that isn't in `SEED_PRICES`/`TICKER_PARAMS` (e.g., the user or the AI assistant adds `PYPL`) starts at a random price in `$50–$300` and uses `DEFAULT_PARAMS`.

## Implementation — `GBMSimulator`

```python
class GBMSimulator:
    """GBM simulator for correlated stock prices."""

    TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600
    DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR

    def __init__(self, tickers, dt=DEFAULT_DT, event_probability=0.001):
        # per-ticker state: self._prices, self._params
        # self._cholesky: np.ndarray | None, rebuilt on ticker add/remove

    def step(self) -> dict[str, float]:
        """Advance every tracked ticker by one dt. Hot path -- called every 500ms."""
        z_independent = np.random.standard_normal(n)
        z_correlated = self._cholesky @ z_independent if self._cholesky is not None else z_independent
        for i, ticker in enumerate(self._tickers):
            mu, sigma = self._params[ticker]["mu"], self._params[ticker]["sigma"]
            drift = (mu - 0.5 * sigma**2) * self._dt
            diffusion = sigma * math.sqrt(self._dt) * z_correlated[i]
            self._prices[ticker] *= math.exp(drift + diffusion)
            if random.random() < self._event_prob:
                shock = random.uniform(0.02, 0.05) * random.choice([-1, 1])
                self._prices[ticker] *= (1 + shock)
            result[ticker] = round(self._prices[ticker], 2)
        return result

    def add_ticker(self, ticker) -> None: ...     # seeds price/params, rebuilds Cholesky
    def remove_ticker(self, ticker) -> None: ...  # drops state, rebuilds Cholesky
    def get_price(self, ticker) -> float | None: ...
    def get_tickers(self) -> list[str]: ...
```

`step()` is the hot path (every 500ms, for every connected use of the app), so it's kept allocation-light: one `numpy` draw for all tickers at once, one matrix multiply, then a plain Python loop over a handful of tickers (watchlists are small — tens, not thousands).

## `SimulatorDataSource` — the `MarketDataSource` adapter

```python
class SimulatorDataSource(MarketDataSource):
    def __init__(self, price_cache, update_interval=0.5, event_probability=0.001): ...

    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(tickers=tickers, event_probability=self._event_prob)
        # Seed the cache immediately -- SSE clients get data before the first tick,
        # rather than an empty watchlist for the first 500ms.
        for ticker in tickers:
            self._cache.update(ticker=ticker, price=self._sim.get_price(ticker))
        self._task = asyncio.create_task(self._run_loop(), name="simulator-loop")

    async def _run_loop(self) -> None:
        while True:
            try:
                for ticker, price in self._sim.step().items():
                    self._cache.update(ticker=ticker, price=price)
            except Exception:
                logger.exception("Simulator step failed")   # never let one bad step kill the loop
            await asyncio.sleep(self._interval)
```

Two behaviors worth noting:
- **Immediate cache seeding in `start()`** — without this, a client connecting right at startup would see an empty watchlist for up to 500ms.
- **`_run_loop` swallows and logs exceptions per iteration** rather than propagating — a transient failure in `step()` (there shouldn't be one, but e.g. a NaN from a pathological input) degrades to "prices stop updating momentarily" rather than crashing the background task and permanently freezing every price in the app.

## File Structure

```
backend/
  app/
    market/
      simulator.py       # GBMSimulator + SimulatorDataSource
      seed_prices.py      # SEED_PRICES, TICKER_PARAMS, DEFAULT_PARAMS, correlation constants
```

`seed_prices.py` holds only constant dictionaries/sets so they're trivially testable and tweakable without touching simulation logic; `simulator.py` holds the two classes.

## Behavior Notes

- Prices can't go negative — GBM is multiplicative (`exp()` output is always positive), unlike an additive random walk.
- With `sigma=0.50` (TSLA) and the default `dt`, a full trading day of ticks produces roughly the intraday range you'd expect from a real high-volatility name.
- A correlation matrix built from `{0.6, 0.5, 0.3}` pairwise blocks is guaranteed positive semi-definite for these fixed constants, so `np.linalg.cholesky` never raises — this would need re-checking if the correlation values or grouping logic changes.
- Rebuilding the Cholesky decomposition on every `add_ticker`/`remove_ticker` is `O(n^2)` to build the matrix plus `O(n^3)` for the decomposition, but `n` (watchlist size) stays small in this single-user app, so this is a non-issue in practice.
- `TICKER_PARAMS`/`SEED_PRICES` are last verified against real market prices "as of project creation" — they're cosmetic seed values for the simulated world, not live reference data, and don't need to track real prices over time.
