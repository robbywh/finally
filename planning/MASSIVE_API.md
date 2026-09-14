# Massive API Reference (formerly Polygon.io)

Reference documentation for the Massive REST API as used in FinAlly, verified against live docs on 2026-09-14. This supersedes `planning/archive/MASSIVE_API.md`, which predates the rebrand-era plan-tier details below and contained one incorrect field name (`day.previous_close` — see [Corrections](#corrections-vs-the-archived-version)).

## Overview

- **Rebrand**: Polygon.io became Massive on **2025-10-30**. Same company, same data, same accounts — APIs, SDKs, and existing endpoints continue to work unchanged. Both `api.polygon.io` and `api.massive.com` operate in parallel during the migration window. ([Massive blog](https://massive.com/blog/polygon-is-now-massive), [FISD](https://fisd.net/polygon-io-is-now-massive/))
- **Base URL**: `https://api.massive.com` (legacy `https://api.polygon.io` still works)
- **Python package**: [`massive`](https://pypi.org/project/massive/) on PyPI — the renamed successor to `polygon-api-client`. Install: `uv add massive` / `pip install -U massive`. Source: [massive-com/client-python](https://github.com/massive-com/client-python).
- **Auth**: API key via the `MASSIVE_API_KEY` environment variable (read automatically by `RESTClient()`), or passed explicitly: `RESTClient(api_key="...")`. Internally sent as an `Authorization: Bearer <API_KEY>` header.

## Plan Tiers & Rate Limits — read this before designing the poller

This is the most important thing to get right, and the part earlier internal notes got wrong. Access is **not** just about rate limits — different endpoints are gated by plan tier entirely:

| Plan | Cost | Request limit | Snapshot endpoints | Aggregates / previous-close | History | Recency |
|------|------|----------------|---------------------|------------------------------|---------|---------|
| **Basic (free)** | $0/mo | 5 req/min | **Not included** | Included | 2 years | End-of-day only |
| **Starter** | $29/mo | Unlimited | Included | Included | 5 years | 15-min delayed |
| **Developer** | $79/mo | Unlimited | Included | Included | 10 years | 15-min delayed |
| **Advanced** | $199/mo | Unlimited | Included | Included | 20+ years | Real-time |
| **Business** | custom | Unlimited | Included | Included | All | Real-time |

Sources: [request-limit FAQ](https://massive.com/knowledge-base/article/what-is-the-request-limit-for-massives-restful-apis), [Full Market Snapshot plan table](https://massive.com/docs/rest/stocks/snapshots/full-market-snapshot), [Daily Market Summary plan table](https://massive.com/docs/rest/stocks/aggregates/daily-market-summary).

**Practical implication for FinAlly:**

- A **free Basic-tier key cannot call the snapshot endpoints at all** — they 403. The free tier only unlocks end-of-day aggregates (previous close, grouped daily).
- To get intraday/"real-time" polling behavior (what the plan's live watchlist needs), the user's `MASSIVE_API_KEY` needs to be at least a **Starter** plan, and even then data is **15-minute delayed** until Advanced/Business.
- Paid tiers have no hard rate cap, but Massive asks integrators to stay under ~100 req/s of sustained load ("we do monitor usage to ensure that no single user affects the quality of service for others").
- Document this to end users of FinAlly: plugging in a free-tier key will not produce live watchlist updates — the simulator is a better default in that case. See [MARKET_INTERFACE.md](MARKET_INTERFACE.md) for how the factory should treat this.

## Client Initialization

```python
from massive import RESTClient

# Reads MASSIVE_API_KEY from environment automatically
client = RESTClient()

# Or pass explicitly
client = RESTClient(api_key="your_key_here")

# Full constructor (defaults shown)
client = RESTClient(
    api_key="your_key_here",
    api_base="https://api.massive.com",
    pagination=True,
    verbose=False,
    trace=False,
)
```

## Endpoints Used in FinAlly

### 1. Full Market Snapshot — Multiple Tickers (real-time/delayed polling)

The main endpoint for the live watchlist. Gets current prices for a comma-separated set of tickers in **one API call**. **Requires Starter plan or higher** (see table above).

**REST**: `GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,GOOGL,MSFT`

**Python client**:
```python
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient()

snapshots = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA"],
)

for snap in snapshots:
    print(f"{snap.ticker}: ${snap.last_trade.price}")
    print(f"  Day change: {snap.todays_change_percent}%")
    print(f"  Day OHLC: O={snap.day.open} H={snap.day.high} L={snap.day.low} C={snap.day.close}")
    print(f"  Prev close: {snap.prev_day.close}")
```

**Signature** (from the client source):
```python
def get_snapshot_all(
    self,
    market_type: Union[str, SnapshotMarketType],
    tickers: Optional[Union[str, List[str]]] = None,
    params: Optional[Dict[str, Any]] = None,
    raw: bool = False,
    include_otc: Optional[bool] = False,
    options: Optional[RequestOptionBuilder] = None,
) -> Union[List[TickerSnapshot], HTTPResponse]
```

**Raw JSON response** (per ticker, camelCase on the wire):
```json
{
  "ticker": "AAPL",
  "day": { "o": 129.61, "h": 130.15, "l": 125.07, "c": 125.07, "v": 111237700, "vw": 127.35 },
  "prevDay": { "o": 128.0, "h": 130.0, "l": 127.5, "c": 129.61, "v": 98000000, "vw": 128.9 },
  "min": { "t": 1675190399000, "o": 125.0, "h": 125.1, "l": 124.9, "c": 125.07, "v": 12000, "n": 40 },
  "lastTrade": { "p": 125.07, "s": 100, "x": 11, "t": 1675190399000000000, "i": "12345" },
  "lastQuote": { "p": 125.06, "P": 125.08, "s": 500, "S": 1000, "t": 1675190399500000000 },
  "todaysChange": -4.54,
  "todaysChangePerc": -3.50,
  "updated": 1675190399000000000,
  "fmv": 125.05
}
```

**Field name mapping — Python client attributes are snake_case and do NOT literally mirror the JSON tree** (this is where the earlier internal notes were wrong):

| Python attribute | JSON key | Notes |
|---|---|---|
| `snap.ticker` | `ticker` | string |
| `snap.day.open/high/low/close/volume/vwap` | `day.o/h/l/c/v/vw` | current session's aggregate bar |
| `snap.prev_day.open/high/low/close/volume/vwap` | `prevDay.o/h/l/c/v/vw` | **previous close lives here, as `prev_day.close` — there is no `day.previous_close`** |
| `snap.last_trade.price` (`.p`), `.timestamp` (`.t`, **nanoseconds**), `.size` (`.s`) | `lastTrade.*` | requires trade-level access on your plan |
| `snap.last_quote.bid_price`/`ask_price` | `lastQuote.p`/`P` | note capitalization: lowercase `p` = bid, uppercase `P` = ask |
| `snap.todays_change` / `snap.todays_change_percent` | `todaysChange` / `todaysChangePerc` | precomputed day change — cheaper than recomputing from `prev_day.close` |
| `snap.fair_market_value` | `fmv` | Business plan only |

**Timestamps**: `lastTrade.t` / `lastQuote.t` / `updated` are **Unix nanoseconds**, not milliseconds — divide by `1e9` for seconds (aggregate bar timestamps like `day.t`/`min.t` are milliseconds; be careful not to conflate the two).

**Data lifecycle**: snapshot data resets daily around 3:30 AM ET and repopulates from ~4:00 AM ET pre-market.

### 2. Single Ticker Snapshot

For a detail view of one ticker. Same plan restriction as above (Starter+).

```python
snapshot = client.get_snapshot_ticker(
    market_type=SnapshotMarketType.STOCKS,
    ticker="AAPL",
)
print(f"Price: ${snapshot.last_trade.price}")
print(f"Bid/Ask: ${snapshot.last_quote.bid_price} / ${snapshot.last_quote.ask_price}")
print(f"Day range: ${snapshot.day.low} - ${snapshot.day.high}")
```

**REST**: `GET /v2/snapshot/locale/us/markets/stocks/tickers/{ticker}`

### 3. Grouped Daily Bars — End-of-Day for ALL Tickers, One Call (works on free tier)

This is the endpoint to reach for when the requirement is specifically **end-of-day prices across many tickers** and a free/Basic key is in play — it's included on every plan (with recency capped at end-of-day on Basic).

**REST**: `GET /v2/aggs/grouped/locale/us/market/stocks/{date}`

**Python client**:
```python
grouped = client.get_grouped_daily_aggs(
    "2026-09-11",          # trading date, YYYY-MM-DD
    adjusted=True,
    include_otc=False,
)
for bar in grouped:
    print(f"{bar.ticker}: close={bar.close} volume={bar.volume}")
```

**Raw response** (per ticker; note `T` for ticker here, distinct from other endpoints' `ticker`):
```json
{"T": "AAPL", "o": 129.61, "h": 130.15, "l": 125.07, "c": 125.07, "v": 111237700, "vw": 127.35, "t": 1675123200000, "n": 542318}
```

### 4. Previous Close (single ticker EOD)

Included on **every plan, including free**. Useful for seed prices or a lightweight per-ticker EOD check.

**REST**: `GET /v2/aggs/ticker/{ticker}/prev`

```python
prev = client.get_previous_close_agg(ticker="AAPL")
for agg in prev:
    print(f"Previous close: ${agg.close}")
    print(f"OHLC: O={agg.open} H={agg.high} L={agg.low} C={agg.close}  V={agg.volume}")
```

### 5. Aggregates (Custom Bars) — historical OHLCV

For historical charts, not live polling.

**REST**: `GET /v2/aggs/ticker/{ticker}/range/{multiplier}/{timespan}/{from}/{to}`

```python
aggs = []
for a in client.list_aggs(
    ticker="AAPL",
    multiplier=1,
    timespan="day",
    from_="2026-01-01",
    to="2026-01-31",
    limit=50000,
):
    aggs.append(a)

for a in aggs:
    print(f"t={a.timestamp} O={a.open} H={a.high} L={a.low} C={a.close} V={a.volume}")
```

`Agg` attribute mapping: `open/high/low/close/volume/vwap/timestamp/transactions/otc` ← `o/h/l/c/v/vw/t/n/otc`. `timestamp` is **milliseconds** here (unlike snapshot trade/quote timestamps, which are nanoseconds).

### 6. Last Trade / Last Quote

Single most-recent trade or NBBO quote for one ticker — rarely needed once you have the snapshot endpoint, but useful for a cheap single-ticker check.

```python
trade = client.get_last_trade(ticker="AAPL")
print(f"Last trade: ${trade.price} x {trade.size}")

quote = client.get_last_quote(ticker="AAPL")
print(f"Bid: ${quote.bid_price} x {quote.bid_size}")
print(f"Ask: ${quote.ask_price} x {quote.ask_size}")
```

## How FinAlly Uses the API

The Massive poller runs as a background asyncio task (see `backend/app/market/massive_client.py`), mirroring the pattern below:

```python
import asyncio
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

async def poll_massive(api_key: str, get_tickers, price_cache, interval: float = 15.0):
    """Poll Massive's snapshot endpoint and update the shared price cache."""
    client = RESTClient(api_key=api_key)

    while True:
        tickers = get_tickers()
        if tickers:
            # RESTClient is synchronous -- run it off the event loop
            snapshots = await asyncio.to_thread(
                client.get_snapshot_all,
                market_type=SnapshotMarketType.STOCKS,
                tickers=tickers,
            )
            for snap in snapshots:
                price_cache.update(
                    ticker=snap.ticker,
                    price=snap.last_trade.price,
                    timestamp=snap.last_trade.timestamp / 1_000_000_000,  # ns -> s
                )
        await asyncio.sleep(interval)
```

FinAlly deliberately does **not** rely on `prev_day.close` / `todays_change_percent` from the snapshot — the shared `PriceCache` computes `previous_price`/`change`/`direction` itself from the last cached tick (see [MARKET_INTERFACE.md](MARKET_INTERFACE.md)). This keeps the cache's notion of "change" consistent between the simulator and Massive sources, and sidesteps the snapshot's own day-change fields resetting at the 3:30 AM ET data-clear boundary.

## Error Handling

The client raises exceptions for HTTP errors:
- **401**: Invalid API key
- **403**: Plan doesn't include the endpoint (e.g., a Basic/free key calling a snapshot endpoint)
- **429**: Rate limit exceeded (free tier: 5 req/min)
- **5xx**: Server errors (client has built-in retry, 3 attempts by default)

A poller should catch and log these per cycle rather than crash the background task — a transient 429/5xx should just be retried on the next interval (see `MassiveDataSource._poll_once` in the codebase, which does exactly this).

## Corrections vs. the archived version

`planning/archive/MASSIVE_API.md` (the version written before this refresh) got two things wrong that this document fixes:

1. **`day.previous_close` does not exist.** Previous close is a separate object: `prev_day.close` (Python) / `prevDay.c` (JSON). The current `massive_client.py` implementation doesn't use this field at all (it derives `previous_price` from the cache instead), so the bug never shipped — but the doc itself was wrong and would have misled anyone implementing against it directly.
2. **No mention of the snapshot endpoint's plan gating.** The archive doc treated the free tier as merely rate-limited (5 req/min) with full endpoint access. In fact the free/Basic tier cannot call snapshot endpoints at all — it only has aggregate/previous-close access. This is now documented above and should inform user-facing setup instructions (e.g., a README note: "a free Massive key will not enable live prices; get a Starter+ key or use the built-in simulator").

## Sources

- [Polygon.io is Now Massive (official announcement)](https://massive.com/blog/polygon-is-now-massive)
- [Polygon.io is Now Massive – FISD](https://fisd.net/polygon-io-is-now-massive/)
- [`massive` on PyPI](https://pypi.org/project/massive/)
- [massive-com/client-python (GitHub)](https://github.com/massive-com/client-python)
- [Stocks REST API overview](https://massive.com/docs/rest/stocks/overview)
- [Full Market Snapshot docs](https://massive.com/docs/rest/stocks/snapshots/full-market-snapshot)
- [Single Ticker Snapshot docs](https://massive.com/docs/rest/stocks/snapshots/single-ticker-snapshot)
- [Daily Market Summary (grouped daily) docs](https://massive.com/docs/rest/stocks/aggregates/daily-market-summary)
- [Previous Day Bar docs](https://massive.com/docs/rest/stocks/aggregates/previous-day-bar)
- [Custom Bars (aggregates) docs](https://massive.com/docs/rest/stocks/aggregates/custom-bars)
- [Request limit FAQ](https://massive.com/knowledge-base/article/what-is-the-request-limit-for-massives-restful-apis)
- [Pricing](https://massive.com/pricing)
