# Hosted service API contract (integration hooks)

These are proposed interfaces for the hosted backend. They are **not implemented by the static preview**. Keep provider credentials server-side; the browser must never receive broker secrets. Require authenticated users, scoped tokens, TLS, request signing where supported, audit logging, rate limits, and idempotency keys on all order mutations.

## Read APIs

| Method and path | Purpose |
|---|---|
| `GET /api/v1/health` | Service status, provider connection state, feed timestamp and staleness. |
| `GET /api/v1/market/bars?symbol=NIFTY&interval=1m&limit=300` | OHLCV bars, timezone, source, and last complete candle. Intervals: `1m`, `5m`, `15m`. |
| `GET /api/v1/market/chain?underlying=NIFTY&expiry=...` | Calls/puts with strike, bid/ask and sizes, IV, OI, OI change, LTP, timestamp, and source. |
| `GET /api/v1/signals/today` | One-shot result, rule/config version, indicator inputs, rationale, eligibility, red flags, and data freshness. Return `no_trade` when any required condition is unknown or stale. |
| `GET /api/v1/scanner?limit=20` | Ranked alternatives with score breakdown, spread, risk levels, and chart/deep link. |
| `GET /api/v1/analytics/backtest?from=...&to=...&lookback_sessions=180&rule_version=...` | Versioned in-sample and out-of-sample statistics, dataset provenance, confusion matrix, and walk-forward windows. Prefer `lookback_sessions=180` to mean exactly 180 exchange sessions; `from` and `to` bound the eligible dates. |
| `GET /api/v1/risk/estimate?signal_id=...` | Max risk, lots, margin estimate, contract metadata, and broker estimate timestamp. |
| `GET /api/v1/trades?from=...&to=...` | Trade ledger and audit references. |

## Settings and connection APIs

| Method and path | Purpose |
|---|---|
| `GET /api/v1/settings` | Current user's versioned parameters and safety toggles. |
| `PUT /api/v1/settings` | Validate and persist settings; record actor, timestamp, and new version. |
| `POST /api/v1/connections/market-data` | Begin dashboard-managed provider authorization; return an OAuth redirect/status, not a secret. |
| `POST /api/v1/connections/broker` | Begin broker authorization/sandbox selection. Keep refresh/access tokens in a secret store. |
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

Required response fields:

- `as_of`: ISO-8601 quote/signal timestamp; include `source`, `source_timestamp`, `delay_minutes`, and `data_quality` for auditability.
- `eligible`, `action`, `rationale`, `red_flags`, `confidence_pct`, and `safety_mode`.
- `validation`: true/false fields `ema_cross`, `volume_confirmation`, `supertrend`, `spread_limit`, `thin_volume_guard`, `trap_cooldown`, `position_cap`, `stop_cap`, `one_shot_available`; when dashboard safety mode is on, also `same_timeframe_confirmation`.
- `contract`: `strike`, `type` (`CE`/`PE`), `entry_premium`, `bid`, `ask`, `bid_size`, `ask_size`, `oi`, `oi_change`, and `iv_percentile`.
- `stop_points` (must be positive and at most 5), `target_points` (15–20 for the primary signal), and `position_spend` (must be at most ₹10,000). `indicators` should provide EMA cross, volume ratio, SuperTrend direction, spread percentage, and OI change.

`GET /api/v1/market/chain` should return `{as_of, source, rows:[{strike, call:{bid,ask,bid_size,ask_size,iv,oi,oi_change,ltp}, put:{...}}]}`. The scanner returns `{as_of, items:[{eligible, validation:{all_passed}, strike, option_type, premium, bid, ask, bid_size, ask_size, oi_change, iv_percentile, spread_pct, stop_points, position_spend, expected_points (10–15), historical_win_rate, historical_sample_size, confidence_pct, status}]}`. The browser rejects candidates without a fresh parent timestamp, explicit validation, and all configured caps. Candidate ranking and all expected-point figures must be backed by out-of-sample measurements, not generated from indicator scores alone.

The UI stores editable parameters in this browser and sends `PUT /api/v1/settings` when a hosted API URL is configured. The API should return the persisted settings version and reject unsupported/inconsistent values. Do not put exchange or broker secrets in browser JavaScript.

## Source availability

The dashboard's TradingView widget is a visual chart only; its values must not be read, scraped, or used by the rule engine. TradingView documents widgets as delayed/display-only and says it does not expose a data API. Investing.com says it has no public API. Yahoo's terms restrict automated collection without prior permission. NSE's official delayed snapshot product is subscription-priced, and its data policy governs subscriber use. Google Finance does not provide a supported public API suitable for this dashboard. Use an authorized provider endpoint for calculations and show its vendor, license/source, timestamp, and delay.
## Exact one-shot rule handling

The server should only mark `eligible=true` after all strategy filters pass on completed candles and a fresh option quote: a completed 20/200 EMA cross; volume at least 1.5 times the prior five completed bars' average; aligned SuperTrend; spread no greater than 0.5%; a thin-liquidity/spike guard; stop no wider than 5 option points; total position spend no greater than ₹10,000; and no prior primary recommendation that session. The browser separately requires every named validation flag plus `one_shot_available=true`. On an early stop (first 3 minutes), if reversal volume exceeds 2 times its reference average, set `trap_cooldown=false` to indicate an active block and do not issue another entry that session; `true` means no active trap block. Avoid relying on the TradingView iframe for any of these calculations.