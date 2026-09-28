# Signal Desk user guide

## What this preview does

This workspace contains the dashboard front end and a documented service API contract. No public dashboard URL or service deployment is configured yet. The embedded TradingView chart is display-only. Actual quotes, calculated recommendations, scanner rankings, and historical performance require a hosted API connected to an authorized data source.

## One-shot rule

The dashboard's default rule is a proposed rule, not a backtested strategy:

1. On a completed 1-minute candle, EMA(20) crosses EMA(200). A bullish cross is a long setup; a bearish cross is a short setup.
2. Candle volume is at least 1.5× the mean volume of the five preceding completed candles.
3. SuperTrend(10, 3) points in the same direction as the cross.
4. The selected option's bid/ask spread is within the configured maximum, and its premium is below the configured cap.
5. In safety mode, a same-direction setup is also present on 5-minute candles before a recommendation is issued.

Only the first qualifying setup per session is eligible. Avoid spreads above 0.5% and thin-volume spikes. Stop distance is capped at 5 option points; target aim is 15–20 option points and max position spend is ₹10,000. If the stop is hit in the first 3 minutes and reversal volume exceeds 2× average, flag a trap and block re-entry. These rules still require a timestamped options feed and a fixed strike/expiry/fill model before testing.

## Scanner and order workflow

The scanner should rank eligible setups using a published, deterministic score and show each score component. “View” opens that instrument's chart. An order ticket should be prefilled from the validated signal and risk budget. Paper/replay mode must be the default. A live order requires a separate confirmation for each order; turning on a general live-mode switch is not confirmation. Include a server-side daily loss limit, duplicate-order/idempotency protection, stale-quote rejection, and a cancel-all emergency action.

The client does not submit orders. Scanner data only appears after a service returns ranked, validated candidates.

## Backtest and validation

Do not publish performance figures until a versioned dataset and exact options execution rule are available. For a 180-trading-session evaluation, record the data vendor, timezone/session calendar, missing-bar treatment, signal timestamp, next-bar fill model, fees, taxes, spread, slippage, lot sizing, and same-bar stop/target tie handling. Do not use future data in indicators or fills. Report sample size, win rate, average and median points, maximum consecutive losses, max drawdown, and net P/L on ₹100,000 starting capital.

Use chronological rolling 60-session out-of-sample windows. Keep the strategy parameters fixed during each window; if parameters are tuned, tune only on an earlier training window and report those choices. Report every window's sample size and win rate, plus dispersion across windows. Define “expected accuracy” explicitly (for example, balanced accuracy from the confusion matrix); it is not interchangeable with win rate. A signal outcome is target-first versus stop-first under the stated fill model; unresolved or same-bar touches must be handled consistently and reported.

## Emergency stop

In a live deployment, the emergency action should block new entries, cancel all open orders, and show broker acknowledgements. Define whether it also exits open positions; that choice must be explicit. Test it against the broker's paper/sandbox API, including timeouts and partial fills, and keep the broker's own terminal/app available as a fallback. This static preview has no broker connection, so it cannot cancel or exit orders.

## Dashboard tuning

The Settings tab defaults to EMA 20/200, SuperTrend ATR multiplier 3, volume threshold 1.5× the prior five-bar average, max spread 0.5%, max spend ₹10,000, stop 5 option points, and target aim 20 points. Parameters are stored in browser storage and sent to `PUT /api/v1/settings` when connected. Without the hosted evaluator they do not compute a signal.

## Hosted dashboard connection

The dashboard now accepts a hosted Signal Desk service URL in **Settings & connections**. Enter the HTTPS base URL and choose **Connect & load data**. The page checks `/api/v1/health` and `/api/v1/signals/today`; it issues no recommendation when eligibility is false, feed staleness is flagged, the health check fails, or a trustworthy timestamp is absent. Optional scanner, backtest, and bar data load from the documented read routes. The API host must allow browser requests from the dashboard origin (CORS).

This workspace contains the dashboard client and API contract, but no hosted service, provider authorization, market history, or public deployment URL. A service URL and provider authorization therefore need to exist before the dashboard can return actual recommendations. The dashboard does not expose broker credentials or place live orders; a production service must implement and secure the broker routes, and live orders must require per-order confirmation.

## Requested scalping guardrails

The dashboard defaults now state limits in option-premium terms: ₹10,000 maximum position spend, a 5-point option premium stop, and a 15–20 point option premium target aim. A scanner alternate should use the same hard risk checks and may display a 10–15 point target aim. These are user-defined constraints, not evidence that those returns are attainable. A 5-point stop can be hit by spread widening or a fast move; it cannot prevent a loss beyond the planned level in all market conditions. The service must size from live lot size, premium, bid/ask depth, and risk budget and decline trades that cannot meet all limits.

The requested 95% hit rate is not a valid promise or a safe default. Validate any chosen rule on timestamped, out-of-sample data after realistic spread, fees, slippage, and stop fills; show the sample size and confidence interval. Backtests cannot guarantee future results. SEBI reported that 93% of individual equity F&O traders incurred losses over FY22–FY24 ([SEBI study announcement](https://www.sebi.gov.in/media-and-notifications/press-releases/sep-2024/updated-sebi-study-reveals-93-of-individual-traders-incurred-losses-in-equity-fando-between-fy22-and-fy24-aggregate-losses-exceed-1-8-lakh-crores-over-three-years_86906.html)).
## External chart source

The dashboard embeds TradingView's NIFTY chart as a display-only chart. It is not used by the rule engine. TradingView states that its widgets can be delayed, cannot export/download their data, and have no API for retrieving chart data; its data terms also restrict automated/non-display use. Investing.com says it does not provide public API access, while Yahoo's terms restrict automated collection without express prior permission. For computations (signals, backtests, options chain, OI, liquidity, and sizing), connect an exchange-authorized provider through the hosted service API instead.
## GitHub Pages and broker connectivity

The published GitHub Pages URL is a static front end. Recent dashboard updates add a TradingView symbol-editable chart, data-context preferences, broker selection, a secure broker sign-in hand-off, and automatic refresh. The live signal, broker login, options data, and analytics still need an HTTPS Signal Desk API service. The browser never stores broker credentials; the backend needs a provider-specific adapter and secure secret store.

In Settings, choose a data context, max quote age (1–15 minutes), expiry/strike preference, and a supported broker. Enter the Signal service URL and connect it. Broker sign-in redirects to the selected broker only when that server route is implemented. For “Other,” the service must have a custom adapter; typing a broker label does not make the provider API compatible. Options and market-data entitlements vary by broker and account. A pre-open or prior-close-only snapshot is context, not a valid intraday entry trigger; the dashboard will show no trade when the live quote/candle is stale.

The page files have been updated in the workspace, but they are not committed or published from here: this workspace has no Git repository or remote. GitHub Pages will continue to serve its current build until the updated files are deployed to the repository. No credentials or GitHub account changes were made.