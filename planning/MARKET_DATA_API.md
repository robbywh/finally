# Market Data API Design

The HTTP-facing contract for market data, as implemented in `backend/app/market/stream.py`. This is the doc frontend/chat integration should read — it describes the wire format a client actually receives, not the Python module internals (see [MARKET_INTERFACE.md](MARKET_INTERFACE.md) for those, and [MARKET_DATA_SUMMARY.md](MARKET_DATA_SUMMARY.md) for the build overview).

Market data currently exposes exactly one HTTP endpoint, per [PLAN.md §8](PLAN.md#8-api-endpoints). There is no REST endpoint for "get current price of X" — the SSE stream is the only way prices reach a client; portfolio/trade routes read prices server-side straight from the shared `PriceCache` (see `MARKET_INTERFACE.md`), not over HTTP.

## `GET /api/stream/prices`

Server-Sent Events stream of live prices for every ticker currently tracked by the market data source (i.e. the union of the watchlist and any tickers held as open positions — see [Watchlist Coordination](#watchlist-coordination-and-the-tracked-set) below).

### Request

No parameters, no auth, no request body. One connection per browser tab; the frontend opens it once at page load with:

```javascript
const es = new EventSource('/api/stream/prices');
```

### Response

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
X-Accel-Buffering: no
```

`X-Accel-Buffering: no` matters only if the app is ever put behind an nginx reverse proxy — it stops nginx from buffering the stream into unresponsive chunks. It is a no-op running directly behind uvicorn, which is FinAlly's default deployment (see [PLAN.md §11](PLAN.md#11-docker--deployment)), so it costs nothing to leave in.

The very first bytes on the wire are a retry directive, before any data:

```
retry: 1000

```

This tells the browser's native `EventSource` to wait 1s before reconnecting on drop, rather than its own (browser-dependent, often longer) default backoff.

### Event format

Every subsequent event is a single `data:` line — one JSON object per event, keyed by ticker, containing every tracked ticker's latest snapshot:

```
data: {"AAPL":{"ticker":"AAPL","price":190.50,"previous_price":190.42,"timestamp":1707580800.5,"change":0.08,"change_percent":0.042,"direction":"up"},"GOOGL":{"ticker":"GOOGL","price":175.12,"previous_price":175.12,"timestamp":1707580800.0,"change":0.0,"change_percent":0.0,"direction":"flat"}}

```

Per-ticker object shape (this is `PriceUpdate.to_dict()` — see [MARKET_INTERFACE.md §Core Data Model](MARKET_INTERFACE.md#core-data-model--modelspy)):

| Field | Type | Notes |
|---|---|---|
| `ticker` | string | Redundant with the outer object's key; included so a consumer can destructure a single ticker's value without the key |
| `price` | number | Latest price, rounded to 2dp |
| `previous_price` | number | Price as of the previous tick FinAlly observed for this ticker (not the source's own "previous close" — see [MARKET_INTERFACE.md](MARKET_INTERFACE.md#price-cache--cachepy) for why) |
| `timestamp` | number | Unix seconds (float), when this price was recorded into the cache |
| `change` | number | `price - previous_price`, rounded to 4dp |
| `change_percent` | number | Percent form of `change`, rounded to 4dp; `0.0` if `previous_price` is `0` |
| `direction` | string | `"up"` \| `"down"` \| `"flat"` — drives the green/red price-flash CSS per [PLAN.md §2](PLAN.md#visual-design) |

On a ticker's very first appearance in the cache (just added to the watchlist, or the app just started), `previous_price == price` and `direction == "flat"` — there is no meaningful "previous" yet.

**The event is a full snapshot, not a diff.** Every send includes every tracked ticker, even ones that didn't change since the last send (the server only decides *whether* to send based on `PriceCache.version`, not *what* to include — see below). The frontend does not need to merge partial updates; each `event.data` fully replaces its local price table.

### Cadence and change detection

The server polls its internal cache every 500ms but only pushes an event if something changed since the last push:

- **Simulator source**: effectively every tick (500ms), since GBM prices move continuously.
- **Massive source**: only on the poller's own cadence (default 15s on a free-tier-adjacent poll interval; faster on paid tiers — see [MASSIVE_API.md](MASSIVE_API.md#plan-tiers--rate-limits--read-this-before-designing-the-poller)). Between polls, no events are sent at all — this is expected, not a stall.

A consequence for frontend code: **do not assume a fixed event rate.** Sparkline accumulation and "last updated" indicators should key off each event's own `timestamp`, not off wall-clock polling on the client side.

### Empty watchlist

If no tickers are tracked (user cleared the watchlist and holds no positions), the server sends no `data:` events at all — the connection stays open (so the client sees itself as "connected"), but silent, until a ticker is added. This is a real, expected steady state, not an error — see [MARKET_INTERFACE.md §Differences](MARKET_INTERFACE.md) and the archived design's [§13.1](archive/MARKET_DATA_DESIGN.md#131-startup-empty-watchlist).

### Disconnection and reconnection

- The server checks for client disconnect every 500ms (`request.is_disconnected()`) and closes the generator promptly when the browser tab closes or navigates away — it does not wait for a write to fail.
- `EventSource`'s native auto-reconnect handles the client side: on any drop (network blip, backend restart, proxy timeout) the browser reconnects after the `retry: 1000` directive's 1s delay, no client code required.
- A reconnect is a **fresh connection, not a resume** — there is no `Last-Event-ID` support and no event ID field. The very next event a reconnected client receives is a full current snapshot, so no data is permanently lost; only the interval during which the client was disconnected has a gap. This matches the connection-status indicator in [PLAN.md §2](PLAN.md#visual-design) (green/yellow/red dot): yellow while `EventSource.readyState === EventSource.CONNECTING`, i.e. between drop and the next successful event.
- The frontend's sparkline accumulation ([PLAN.md §10](PLAN.md#10-frontend-design)) is client-side and in-memory only — a reconnect does not replay history, so a sparkline's gap during a disconnect is simply a flatter/shorter line, never backfilled.

### Errors

There is no error-response shape for this endpoint beyond standard HTTP — a client cannot receive a 4xx/5xx mid-stream once the `200` response has started. Failure modes instead show up as *silence*:

| Symptom | Cause | Client-visible effect |
|---|---|---|
| No `data:` events ever, connection open | Watchlist is empty | Expected steady state, not an error (see above) |
| No `data:` events ever, connection open | Massive API key invalid/unauthorized (401) or wrong plan tier (403) | Poller logs the failure server-side and keeps retrying; cache stays empty; stream stays "connected" but silent — see [MASSIVE_API.md §Error Handling](MASSIVE_API.md#error-handling) |
| Events for some tickers missing from an otherwise-populated payload | A single Massive snapshot failed to parse | That ticker just doesn't appear in the object this tick; others are unaffected |
| Connection drops, browser shows reconnecting | Server restart, network blip, dev-server reload | `EventSource` auto-reconnects per above; connection-status dot goes yellow, then green on reconnect |

There is no separate `/api/health`-style signal specific to market data — [PLAN.md §8](PLAN.md#8-api-endpoints)'s general `GET /api/health` covers the app process being up, not whether the price feed itself has data. A frontend that wants to distinguish "connected but no data" from "receiving data" must do so locally, from whether it has received any `data:` event yet.

## Watchlist coordination and the tracked set

This endpoint has no query parameters for selecting tickers — it always streams the *entire* currently-tracked set, which is server state, not something the client requests. That set changes only through the watchlist/portfolio routes (`POST /api/watchlist`, `DELETE /api/watchlist/{ticker}`, and trade execution keeping a held position's ticker tracked even if removed from the watchlist — see [MARKET_INTERFACE.md](MARKET_INTERFACE.md) and the archived design's [§11](archive/MARKET_DATA_DESIGN.md#11-watchlist-coordination)), which call `source.add_ticker()`/`source.remove_ticker()` on the shared `MarketDataSource`. The very next stream tick after such a change reflects the new set automatically — no client-side re-subscription needed, since there was never a subscription to begin with, only "give me everything."

## Non-goals

- **No WebSocket variant.** SSE is a deliberate, one-way-only choice per [PLAN.md §3](PLAN.md#why-these-choices) — trade execution and watchlist changes are ordinary request/response REST calls, not sent over this stream.
- **No historical/backfill query on this endpoint.** A ticker's price history for charting comes from `GET /api/portfolio/history` (portfolio snapshots) and, for the per-ticker detail chart, from whatever the frontend has accumulated client-side since page load — see [PLAN.md §2](PLAN.md#what-the-user-can-do). There is no `GET /api/prices/{ticker}/history` endpoint.
- **No per-client filtering.** Because FinAlly is single-user ([PLAN.md §7](PLAN.md#7-database)), "all tracked tickers" and "this user's watchlist + positions" are the same set — there's no multi-tenant need to scope the stream per connection.
