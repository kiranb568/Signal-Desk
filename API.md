# Hosted service API contract (integration hooks)

These are proposed interfaces for the hosted backend. They are **not implemented by the static preview**. Keep provider credentials server-side; the browser must never receive broker secrets. Require authenticated users, scoped tokens, TLS, request signing where supported, audit logging, rate limits, and idempotency keys on all order mutations.

## Read APIs

| Method and path | Purpose |
|---|---|
| `GET /api/v1/health` | Service status, per-provider connection state, feed timestamp, selected data mode, and staleness. |
| `GET /api/v1/market/bars?symbol=NSE:NIFTY&interval=1m&limit=300` | OHLCV bars, timezone, source, and last complete candle. Accept the selected NSE index or equity symbol. Intervals: `1m`, `5m`, `15m`. |
| `GET /api/v1/market/context?symbol=NIFTY&session=today` | Previous session close, pre-open snapshot, gap, and latest selected-instrument observation with separate source timestamps. |
| `GET /api/v1/market/chain?underlying=NIFTY&expiry=...` | Calls/puts with strike, bid/ask and sizes, IV, OI, OI change, LTP, timestamp, and source. Only requested for F&O. |
| `GET /api/v1/signals/today?asset_class=fno&symbol=NSE:NIFTY` | One-shot result, rule/config version, indicator inputs, rationale, eligibility, red flags, and data freshness. `asset_class` is `fno` or `equity`; evaluate only the requested symbol and return `no_trade` when any required condition is unknown or stale. |
| `GET /api/v1/scanner?limit=20&asset_class=fno&symbol=NIFTY` | Ranked F&O alternatives with score breakdown, spread, risk levels, and chart/deep link. Equity discovery is provided by the embedded India screener; it is not ingested as a recommendation. |
| `GET /api/v1/analytics/backtest?from=...&to=...&lookback_sessions=180&rule_version=...&asset_class=fno&symbol=NIFTY` | Versioned in-sample and out-of-sample statistics, dataset provenance, confusion matrix, and walk-forward windows for the selected asset and symbol. Prefer `lookback_sessions=180` to mean exactly 180 exchange sessions; `from` and `to` bound the eligible dates. |
| `GET /api/v1/risk/estimate?signal_id=...` | Max risk, lots, margin estimate, contract metadata, and broker estimate timestamp. |
| `GET /api/v1/trades?from=...&to=...` | Trade ledger and audit references. |

## Settings and connection APIs

| Method and path | Purpose |
|---|---|
| `GET /api/v1/settings` | Current user's versioned parameters and safety toggles. |
| `PUT /api/v1/settings` | Validate and persist settings; record actor, timestamp, and new version. |
| `POST /api/v1/connections/market-data` | Begin dashboard-managed provider authorization; return an OAuth redirect/status, not a secret. |
| `POST /api/v1/connections/broker` | Accept `{provider, provider_label?, return_to, scope:"market_data"}` and return a broker `authorization_url`; perform OAuth code exchange on the server and store tokens encrypted. Support Upstox, Zerodha Kite, DhanHQ, FYERS, Angel One, Alice Blue, Kotak Neo, ICICI Direct, and separately built custom adapters. |
| `DELETE /api/v1/connections/{provider}` | Revoke the provider connection and disable dependent actions. |

## Paper replay and orders

| Method and path | Purpose |
|---|---|
| `POST /api/v1/replay/sessions` | Start replay for a permitted historical interval, data version, rule version, and paper portfolio. |
| `POST /api/v1/replay/sessions/{id}/step` | Advance one event/bar; return signal changes, simulated fills, latency/slippage assumptions, and ledger ID. |
| `POST /api/v1/orders/paper` | Submit an explicitly paper order; require `Idempotency-Key` and signal ID. |
| `POST /api/v1/orders/live/preview` | Validate a proposed live order, quote freshness, risk cap, market status, and broker margin. Never submits. |
| `POST /api/v1/orders/live` | Submit only after a short-lived preview token and per-order user confirmation. Require `Idempotency-Key`. |
| `POST /api/v1/emergency-stop` | Block entries and cancel open orders; return each broker acknowledgement. Position exits must be an explicit separate policy. |

Order requests should include `signal_id`, `config_version`, `instrument`, `side`, `quantity`, `order_type`, `limit_price`, `stop_price`, `target_price`, `mode`, and `Idempotency-Key`. The backend—not the browser—must recompute permitted quantity and risk. The response should return broker/order IDs, accepted/rejected state, timestamps, estimated/actual slippage, and a machine-readable rejection reason.

## Persistence and audit events

Persist immutable signal snapshots, full available tick/quote history references, OHLCV source/version, indicator values, OI changes, spread, settings version, recommendation rationale, user actions, order requests/acks/fills/cancels, replay assumptions, and exit rationale. Store timestamps in UTC and display market timezone. Corrections should append revisions instead of rewriting prior evidence. Define retention and provider licensing limits before storing raw ticks.

## Dashboard integration expectations

The dashboard polls read APIs or consumes an authenticated WebSocket/SSE stream. Show source and timestamp on every quote, and mark the feed stale when the provider's freshness threshold is breached. Fail closed: stale, missing, crossed, or malformed quotes yield no recommendation and disable order actions. A data connection alone does not validate the strategy or activate live orders.

## Dashboard response contract for recommendations

`GET /api/v1/signals/today` must return a timestamped object. The browser displays a trade only when service health is healthy, the feed timestamp is no more than 15 minutes old, `eligible` is `true`, all required validation booleans are explicitly `true`, and both hard caps can be verified. Missing fields fail closed.

Required response fields for both `asset_class` values:

- `as_of`: ISO-8601 quote/signal timestamp; include `source`, `source_timestamp`, `delay_minutes`, and `data_quality` for auditability.
- `eligible`, `action`, `rationale`, `red_flags`, `confidence_pct`, and `safety_mode`.
- `validation`: true/false fields `ema_cross`, `volume_confirmation`, `supertrend`, `spread_limit`, `thin_volume_guard`, `trap_cooldown`, `position_cap`, `stop_cap`, `one_shot_available`; when dashboard safety mode is on, also `same_timeframe_confirmation`.
- `asset_class` and `instrument` must match the request. For F&O, `contract` contains `strike`, `type` (`CE`/`PE`), `entry_premium`, `bid`, `ask`, `bid_size`, `ask_size`, `oi`, `oi_change`, and `iv_percentile`. For equity, return `entry_price` and a `risk_unit` identifying the price/point convention; do not fabricate an option contract.
- `stop_points` and `target_points` are positive, use the declared `risk_unit`, and `position_spend` must be at most ₹10,000. For F&O, stop is at most 5 premium points and target is 15–20 premium points. Equity risk limits must be separately validated by the service and must not be labeled option-premium points. `indicators` should provide EMA cross, volume ratio, SuperTrend direction, spread percentage when available, and OI change for F&O.

`GET /api/v1/market/chain` should return `{as_of, source, rows:[{strike, call:{bid,ask,bid_size,ask_size,iv,oi,oi_change,ltp}, put:{...}}]}`. The scanner returns `{as_of, items:[{eligible, validation:{all_passed}, strike, option_type, premium, bid, ask, bid_size, ask_size, oi_change, iv_percentile, spread_pct, stop_points, position_spend, expected_points (10–15), historical_win_rate, historical_sample_size, confidence_pct, status}]}`. The browser rejects candidates without a fresh parent timestamp, explicit validation, and all configured caps. Candidate ranking and all expected-point figures must be backed by out-of-sample measurements, not generated from indicator scores alone.

The UI stores editable parameters in this browser and sends `PUT /api/v1/settings` when a hosted API URL is configured. The API should return the persisted settings version and reject unsupported/inconsistent values. Do not put exchange or broker secrets in browser JavaScript.

## Source availability

The dashboard's TradingView widgets are visual-only; their values must not be read, scraped, or used by the rule engine. The India screener is for manual equity discovery, not a scored or validated recommendation. TradingView documents widgets as delayed/display-only and says it does not expose a data API. Investing.com says it has no public API. Yahoo's terms restrict automated collection without prior permission. NSE's official delayed snapshot product is subscription-priced, and its data policy governs subscriber use. Google Finance does not provide a supported public API suitable for this dashboard. Use an authorized provider endpoint for calculations and show its vendor, license/source, timestamp, and delay.
## Exact one-shot rule handling

The server should only mark `eligible=true` after all strategy filters pass on completed candles and a fresh option quote: a completed 20/200 EMA cross; volume at least 1.5 times the prior five completed bars' average; aligned SuperTrend; spread no greater than 0.5%; a thin-liquidity/spike guard; stop no wider than 5 option points; total position spend no greater than ₹10,000; and no prior primary recommendation that session. The browser separately requires every named validation flag plus `one_shot_available=true`. On an early stop (first 3 minutes), if reversal volume exceeds 2 times its reference average, set `trap_cooldown=false` to indicate an active block and do not issue another entry that session; `true` means no active trap block. Avoid relying on the TradingView iframe for any of these calculations.
## GitHub Pages and broker setup

GitHub Pages only hosts static assets. It cannot safely perform OAuth code exchanges, keep broker secrets, poll private broker feeds, or serve the `/api/v1/*` routes. Keep `index.html` on Pages, but point **Signal service URL** at a separate HTTPS backend. Configure the broker app keys/secrets and callback URL in that backend's secret store, not in this page, GitHub Pages files, or browser local storage. The browser asks the service to start broker authorization; the service redirects through the broker and handles the callback/token exchange. After the account connects, the service uses that broker adapter for quotes, candles, option chain, and optional user order/position data.

The dashboard can request prior-session + pre-open + delayed intraday context, or previous-session context only. `data_mode`, `max_data_age_minutes`, `expiry_preference`, `strike_preference`, and strategy limits are persisted by `PUT /api/v1/settings`. The service must return explicit timestamps per source and must not pretend previous-session data is current. A pre-open-only snapshot cannot satisfy an intraday entry rule: until a current completed candle and option quote are within the configured max age, return `eligible=false`.

Broker adapters are provider-specific and must be implemented/tested against the provider's official API. Upstox requires a server-side code-to-token exchange and client secret; Dhan individual API tokens expire and its docs state that Data APIs can carry additional charges. Do not promise a “free broker feed” until the account's entitlement and API pricing confirm it. The dashboard's broker picker does not itself create those server adapters.

## Display widgets and feed boundary

The Overview chart and technical-summary panel can display only the currently selected NSE symbol. The Equity & F&O insights tab embeds TradingView's India stock screener and links to the selected underlying's options chain. These are third-party display widgets only; the dashboard does not scrape or import their rows/ratings into a signal. Equity and derivatives recommendation routes remain the hosted service's responsibility and must identify asset class, symbol/underlying, source timestamp, validation fields, and risk units. A widget loading successfully does not mean an order signal is validated.
