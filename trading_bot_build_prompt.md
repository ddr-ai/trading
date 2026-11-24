# Crypto Trading Bot — Build Prompt for AI Engineer

Copy and paste the entire block below into your AI coding agent. Replace bracketed values like `[YOUR_GEMINI_API_KEY]` with your own credentials before running.

---

## Build Prompt

```
You are an expert full-stack engineer and quantitative trading developer. Build me a FULLY AUTONOMOUS live cryptocurrency trading bot with two client versions: (1) a native iOS app packaged via GitHub Actions as an UNSIGNED .ipa / .app bundle for sideloading with esign (no Apple developer certificate, p12, or provisioning profile from me), and (2) a mobile-optimized PWA. Both clients connect to the same backend that executes live trades on the Gemini exchange.

The core requirement is AUTONOMY: the user sets risk preferences ONCE, and the bot does everything else — market analysis, signal generation, position sizing, order placement, stop-loss and take-profit management, trailing stops, breakeven moves, time-based exits, and emergency flattening — without requiring per-trade approval. The user is notified of actions taken, not asked to approve each one.

## 1. Project Overview

- Purpose: Fully autonomous, set-and-forget crypto day trading bot. The bot does the hard work; the user monitors.
- Traded assets: Bitcoin (BTC), Ethereum (ETH), and one additional high-liquidity coin (Litecoin (LTC) or Solana (SOL)). Third coin must meet liquidity and correlation criteria (see Section 4).
- Exchange: Gemini (API v1 and v2 as appropriate). All execution must route through Gemini spot markets.
- Account minimum: Must support trading with as little as $10 USD available.
- Fractional trading: Must support fractional crypto quantities (e.g., 0.001 BTC, 0.01 ETH).
- Frequency: Multiple trades per day (intraday scalping / day trading style), fully automated.
- Core safety rule: Under no circumstances may the account balance go below $0. No leveraged or margin trading. No short selling. Only spot buy/sell with available settled cash.
- Risk model: User sets risk tolerance once via slider (default moderate, approx 1:1.5 to 1:2 risk:reward). The bot enforces hard risk-management guards at all times regardless of market conditions.
- UX goal: Make trading extremely easy. One-tap "Start Autopilot" / "Stop Autopilot." Plain-English status. No jargon required. User does not need to analyze charts or decide when to trade.

## 2. Deliverables

1. Autonomous Backend Service (Node.js/TypeScript or Python) that:
   - Connects to Gemini REST and WebSocket APIs.
   - Authenticates with API key + secret (HMAC-SHA384 signing).
   - Runs the trading strategy loop continuously (polling + WebSocket) to scan for setups multiple times per day.
   - AUTONOMOUS EXECUTION: When a valid setup is detected and all risk guards pass, the bot places the order automatically without user confirmation.
   - Auto position sizing using Kelly-lite method based on user's risk % and ATR-based stop distance.
   - Auto stop-loss and take-profit placement immediately after entry.
   - Auto trade management: trailing stops, breakeven moves, time-based exits, and partial profit-taking if configured.
   - Enforces all risk-management rules (position sizing, stop-loss, take-profit, daily loss limit, $0 floor, max open positions, correlation caps).
   - Exposes a REST API and/or WebSocket for the iOS app and PWA.
   - Stores API keys encrypted at rest; never logs secrets.
   - Supports paper-trading mode (toggle) before live autopilot.
   - Implements nonce locking to prevent Gemini nonce errors across concurrent requests.
   - Implements WebSocket reconnect with exponential backoff and a stale-price kill switch (no tick for 60s pauses new entries).
   - Sends trade notifications (entry, exit, stop hit, take profit, error) to clients via WebSocket/push.

2. Native iOS app:
   - Language: Swift + SwiftUI.
   - Minimum deployment target: iOS 16+.
   - Build & package: GitHub Actions workflow that builds an UNSIGNED .ipa / .app bundle (no signing, no p12, no provisioning profile secrets). Output the .ipa as a workflow artifact for the user to sign with esign themselves.
   - Sideloading: Provide instructions for installing via esign or similar sideloading tool. No App Store submission required.
   - Features: one-tap Autopilot On/Off, risk slider (set once), portfolio dashboard, live price charts, trade feed / activity log (what the bot did and why), settings, emergency stop-all button, performance metrics (win rate, profit factor, max drawdown, fees paid), push notifications for all bot actions.
   - Must work offline for viewing cached data; trades require network.
   - Setup flow: user enters Gemini API key, sets risk slider, taps "Start Autopilot." After that, no further manual input is required for normal operation.

3. Progressive Web App (PWA):
   - Framework: React, Vue, or Svelte with TypeScript (mobile-first responsive).
   - Optimized for mobile browsers (iOS Safari and Chrome/Android).
   - Installable: web app manifest + service worker for offline caching and add-to-home-screen.
   - Same feature set as the iOS app (autopilot toggle, dashboard, charts, activity log, emergency stop, performance metrics).
   - Communicates with the same backend API.

4. Documentation:
   - .env.example with all required configuration variables.
   - Setup guide for Gemini API key creation (what permissions to enable/disable).
   - GitHub Actions workflow files for iOS unsigned .ipa build.
   - Sideloading instructions for the iOS app (esign).
   - Autopilot explainer: what the bot does automatically, what the user controls, what the guards do.
   - Validation guide: paper trading, walk-forward backtest, Monte Carlo requirements before enabling live autopilot.
   - Disclaimer: live trading involves risk of loss; this is not financial advice.

## 3. Gemini Exchange Integration

- Base URL: https://api.sandbox.gemini.com (paper/test) and https://api.gemini.com (live).
- Authentication: Gemini uses API key + API secret. All private endpoints require a JSON payload signed with HMAC-SHA384, including a nonce.
- Required endpoints to implement:
  - GET /v1/symbols (list tradable symbols, e.g., btcusd, ethusd, ltcusd)
  - GET /v1/pubticker/{symbol} (ticker/price)
  - GET /v1/book/{symbol}?limit_bids=10&limit_asks=10 (order book depth for spread and liquidity checks)
  - GET /v1/balances (account balances)
  - POST /v1/order/new (place order — supports fractional amounts, post-only/maker-only option)
  - GET /v1/orders (active orders)
  - POST /v1/order/cancel/{order_id} or /v1/order/cancel/all (emergency stop)
  - GET /v1/trades/{symbol} (recent trades for strategy signals)
  - WebSocket /v1/marketdata/{symbol} (live price updates for UI and strategy)
- Symbols to trade: btcusd, ethusd, and [ltcusd or solusd] — confirm availability on Gemini before coding.
- Order types: Prefer limit orders with post-only (maker-only) to reduce fees; use market orders only for emergency exits. Do NOT use leverage.
- Order chunking: For orders > $1,000, slice into 2–3 child orders spaced 30 seconds apart to avoid moving the book.
- Fractional amounts: Gemini supports fractional crypto; pass exact decimal quantities (respect symbol-specific step sizes/minimums).
- Nonce management: Use an incrementing, thread-safe nonce source (e.g., Redis or database lock) so concurrent requests never reuse a nonce.

## 4. Autonomous Trading Strategy (Regime-Aware, Multi-Timeframe Confluence)

Implement a rule-based intraday strategy that runs fully autonomously. Do NOT use ML black boxes. Every trade decision must be logged with reason, entry price, stop, target, position size, risk amount, and regime classification.

A. Trend filter (multi-timeframe)
   - Only take long-biased entries when 1h EMA(21) > 1h EMA(50) AND 15m EMA(9) > 15m EMA(21).
   - For stablecoin cash ("short" equivalent), require the inverse.

B. Entry trigger (auto-detected)
   - Mean-reversion bounce off recent support OR momentum breakout above recent resistance.
   - Confirmed by RSI(14) not in extreme overbought (>70) or oversold (<30) territory.
   - Volume confirmation: entry candle volume > 1.5x the 20-period average volume.

C. Volatility regime filter (ATR-based) — auto-skip
   - Skip trades when ATR(14) is in the bottom 20th percentile (dead market) OR top 90th percentile (chaotic/news spike). Trade the middle range.
   - Stop distance = max(ATR(14) × 1.5, 1.5% of entry price).

D. Order book microstructure (Gemini L2) — auto-check
   - Before entering, check spread and depth. If bid/ask spread is wide or depth within 1% of mid is thin (< $5,000 for BTC/ETH), skip or reduce size.
   - Avoid crossing the spread on illiquid coins.

E. Time filter — auto-enforced
   - Avoid trading in the first 15 minutes after major candle closes.
   - Avoid low-volume UTC hours (approximately 02:00–06:00 UTC) unless volume criteria are met.

F. Frequency & cooldown — auto-enforced
   - Max one setup per symbol per 4 hours.
   - Max trades per day configurable (default 3–6 per symbol per day).
   - Cooldown: minimum minutes between trades on the same symbol.

G. Third coin selection criteria (auto-validated at startup)
   - 24h volume on Gemini > $5M.
   - Spread consistently < 0.1%.
   - 30-day correlation to BTC < 0.85. If correlation is higher, do not trade the third coin; trade BTC/ETH only with tighter sizing.

Strategy parameters should be configurable via environment variables. Default to moderate aggression. The user does not tune these; they set risk tolerance only.

## 5. Autonomous Risk Management (Hard Requirements — Never Below $0)

Implement these as absolute guards that override the strategy and run automatically:

a) Account floor: Never place an order if available USD balance < $10. Never allow projected available balance after fees to go below $0.
b) Position sizing (Kelly-lite, automatic): Risk per trade = fixed % of available equity (default 1–2%, hard cap 3%). Position size (crypto amount) = RiskAmount / (EntryPrice − StopPrice). Round down to symbol step size. The bot calculates this automatically; the user does not enter order sizes.
c) Volatility-adjusted stop (automatic): Stop distance = max(ATR(14) × 1.5, 1.5% of price), capped so dollar risk never exceeds the per-trade risk %.
d) Risk-to-reward slider (user sets once):
   - UI slider from 0.5 (conservative) to 3.0 (aggressive), default 1.5.
   - The slider sets the take-profit multiple relative to stop distance ONLY.
   - Take-profit = Entry ± (StopDistance × SliderValue).
   - The slider must NEVER increase position size beyond the per-trade risk cap.
e) Fee & slippage buffer: Subtract estimated Gemini taker/maker fee (0.05%–0.4% depending on tier) + 2 bps slippage from the reward target. If risk:reward after fees < 1:1.2, abort the trade automatically.
f) Daily loss limit: If realized + unrealized P&L for the day hits -3% of starting equity, halt all new trades for the day. At -5%, flatten all positions and halt for the day.
g) Circuit breaker: If max drawdown for the day hits -5%, emergency flatten all positions and disable autopilot until manual reset.
h) Max open positions: Max 1 position per symbol, max 3 concurrent positions total.
i) Buying power cap: Never commit more than 50% of available USD to open positions at once; keep dry powder.
j) Correlation cap: If holding BTC and ETH, block new entries on the third coin if it is > 0.8 correlated to either.
k) Emergency stop: One-tap "Stop All / Flatten" that cancels all open orders, closes all positions at market (or post-only limit if safe), then halts autopilot.
l) No leverage, no margin, no shorting, no options/futures. Spot only.

## 6. Auto Trade Management (Bot-Managed Exits)

The bot manages open positions without user input:

- Immediate stop-loss and take-profit orders placed on Gemini right after entry.
- Trailing stop: after price moves 1× stop distance in favor, trail stop at 0.5× ATR below price (long) or above price (short/cash).
- Breakeven: after price moves 1× stop distance in favor, move stop to entry price (+ small buffer for fees).
- Time exit: if a position is open for more than 4 hours without hitting stop or target, evaluate for exit based on trend filter; if trend flips, exit at market.
- Partial profit: optional — take 50% off at 1× risk, move stop to breakeven, let remainder run.
- All exits are logged and reported to the user via activity feed and push notification.

## 7. Autonomy & User Control Model

- User sets ONCE: Gemini API key, risk slider (0.5–3.0), paper vs live mode, notification preferences.
- User does NOT: pick entry prices, calculate position sizes, place individual orders, approve trades, or manage stops/targets.
- User CAN: turn Autopilot on/off, adjust risk slider (affects future trades only), trigger emergency stop, view activity log, withdraw funds from Gemini directly (bot does not control withdrawals).
- The bot NOTIFIES the user of every action: entry filled, stop adjusted, target hit, position closed, circuit breaker triggered, autopilot paused. Notifications are informational, not approval requests.
- "Set and forget" mode: after starting autopilot, the bot runs 24/7 (within trading time filters) managing entries and exits autonomously.

## 8. Client Apps (iOS + PWA)

Shared UI requirements:
- Landing: Connect Gemini API key (key + secret stored securely: iOS Keychain, PWA encrypted IndexedDB or in-memory only with optional persistence toggle).
- Main screen: Big Autopilot toggle (On/Off), risk slider, status text (e.g., "Autopilot: Scanning…", "Position open: BTC/USD long, target $X, stop $Y"), emergency stop button.
- Dashboard: Account balance, available cash, today's P&L, open positions with auto-managed stops/targets, risk slider, performance metrics (win rate, profit factor, max drawdown, fees paid today).
- Watchlist: BTC/USD, ETH/USD, [third coin]/USD with live prices via WebSocket.
- Activity feed / Trade log: Chronological list of everything the bot did — signals scanned, entries placed, stops moved, exits hit, skips with reason (e.g., "Skipped ETH: low volume").
- Trade history: List of executed trades with entry, exit, P&L, status, and reason code.
- Settings: Paper vs Live toggle, max trades/day, risk %, daily loss limit, notification preferences.
- Alerts: Push notifications for every fill, stop hit, take-profit, daily circuit breaker trigger, autopilot pause/resume, and API disconnect.
- Weekly reset: Reset daily trade counters and P&L tracking at 00:00 UTC.
- Admin kill switch: One button that cancels all open orders, flattens all positions, and sets BOT_ENABLED=false in the backend.
- All screens must be mobile-optimized (thumb-friendly, large touch targets, legible at small screen widths).

iOS specifics:
- SwiftUI, support light/dark mode, portrait/landscape on iPhone.
- Secure storage via Keychain for API credentials.
- Background fetch optional for price updates; trading only when app/backend is active.
- GitHub Actions: build an unsigned .ipa / .app bundle (no signing, no p12, no provisioning profile secrets). Output the .ipa as a workflow artifact for the user to sign with esign themselves.

PWA specifics:
- Responsive mobile-first CSS, no desktop-only layouts.
- Web app manifest (name, short_name, start_url, display: standalone, theme_color, icons).
- Service worker caches static assets and API responses for offline viewing.
- Install prompt (beforeinstallprompt) for Add to Home Screen.

## 9. Security

- API keys: encrypt at rest (AES-256 or platform secure store). Never commit keys to git.
- Use Gemini API keys with only the permissions needed: "Account Balances" (view), "Orders" (trade). Withdrawal MUST BE DISABLED. Advise user to disable withdrawals on the Gemini key.
- All client-to-backend traffic over HTTPS/WSS.
- Input validation on all order parameters.
- Rate-limit respect for Gemini API (avoid 429s).
- Nonce locking for concurrent API requests.
- Paper-trading mode defaults to ON; user must explicitly switch to LIVE.

## 10. Configuration (.env.example)

```
GEMINI_API_KEY=your_key
GEMINI_API_SECRET=your_secret
GEMINI_BASE_URL=https://api.sandbox.gemini.com  # switch to api.gemini.com for live
TRADING_SYMBOLS=btcusd,ethusd,ltcusd
RISK_PER_TRADE_PCT=1.5
MAX_RISK_PER_TRADE_PCT=3.0
DAILY_LOSS_WARNING_PCT=3.0
DAILY_LOSS_LIMIT_PCT=5.0
MAX_CONCURRENT_POSITIONS=3
MAX_TRADES_PER_SYMBOL_PER_DAY=5
MAX_SETUPS_PER_SYMBOL_PER_4H=1
DEFAULT_RISK_REWARD=1.5
RISK_REWARD_MIN=0.5
RISK_REWARD_MAX=3.0
ATR_STOP_MULTIPLIER=1.5
MIN_STOP_PCT=1.5
MAX_BUYING_POWER_PCT=50.0
CORRELATION_CAP=0.85
MIN_VOLUME_MULTIPLIER=1.5
ORDER_CHUNK_THRESHOLD_USD=1000
TRAILING_STOP_ENABLED=true
BREAKEVEN_ENABLED=true
TIME_EXIT_HOURS=4
PAPER_TRADING=true
PORT=3000
ENCRYPTION_KEY=32-byte-random-base64
```

## 11. Validation & Go-Live Requirements (Mandatory)

Do NOT enable live autopilot until all of the following are complete:

1. Paper trading (recommended): Run PAPER_TRADING=true on Gemini sandbox to validate the strategy before going live. Log every would-be trade with entry, stop, target, and result.
2. Walk-forward backtest: Train strategy parameters on 30 days of historical data, validate on the next 7 days, roll forward. Report win rate, avg win/loss, max drawdown, profit factor.
3. Monte Carlo simulation: Randomize trade order 1,000 times to see worst-case drawdown sequences. If any simulation blows the account or breaches the $0 floor, reduce position size.
4. Fee-inclusive backtest: All backtests must include Gemini taker/maker fees + 2 bps slippage.
5. Go-live gate: Only switch PAPER_TRADING=false after the above pass and after the user explicitly confirms live mode.

## 12. Acceptance Criteria

- [ ] Bot connects to Gemini and displays real-time BTC, ETH, and third coin prices.
- [ ] Bot shows account balance and available cash; blocks trades below $10 available.
- [ ] Bot autonomously detects a setup, calculates position size, places entry, stop, and target on Gemini without user confirmation.
- [ ] Bot manages open positions automatically (trailing stop, breakeven, time exit).
- [ ] Risk slider adjusts take-profit while stop-loss and position size respect risk caps.
- [ ] Emergency stop cancels orders and closes positions.
- [ ] Account balance never goes below $0 in paper trading, backtest, or live simulation.
- [ ] Volatility regime filter skips trades in dead or chaotic markets.
- [ ] Order book depth check prevents entries in thin markets.
- [ ] Daily circuit breaker halts at -3% and flattens at -5%.
- [ ] iOS app builds via GitHub Actions and produces an unsigned .ipa for esign sideloading.
- [ ] PWA installs on iOS Safari and Android Chrome via Add to Home Screen.
- [ ] All secrets excluded from version control (.env in .gitignore).
- [ ] Paper trading log generated (recommended before live).
- [ ] Walk-forward backtest and Monte Carlo results documented.
- [ ] User receives push notifications for every autonomous bot action.

## 13. Non-Goals (Do Not Build)

- No App Store submission or TestFlight.
- No leverage/margin/futures/options.
- No withdrawal functionality from Gemini.
- No custodial holding of user funds; funds stay in the user's Gemini account.
- No guaranteed profits or win-rate promises; include clear risk disclaimers.
- No per-trade user approval required in autopilot mode.

Do NOT ask for Apple developer certificates, provisioning profiles, p12 files, or any Apple signing credentials. I will handle signing and sideloading myself using a tool like esign. Build the iOS app as an unsigned .ipa / .app bundle (or provide an Xcode project and export options) that I can sign and install via esign. Proceed to scaffold the full project structure, configuration files, workflows, strategy logic, autonomous risk-management module, auto trade management, and both client apps now. You may ask for Gemini API credentials only.
```

---

## How to Use This Prompt

1. Save the block above as `trading_bot_build_prompt.md`.
2. Open your AI coding agent (e.g., Copilot, Cursor, Claude Code, etc.).
3. Paste the prompt and provide your Gemini API key + secret when asked.
4. The AI will generate:
   - A fully autonomous backend service (Node/TypeScript or Python) for Gemini execution + risk engine + auto trade management.
   - A SwiftUI iOS app + GitHub Actions unsigned build workflow.
   - A mobile PWA with manifest + service worker.
   - Setup, sideloading, and validation instructions.

## Important Warnings

- **Live trading risk:** This bot trades real money automatically without per-trade approval. There is no guarantee of profit; you can lose your entire $10+ account. Use paper-trading mode (`PAPER_TRADING=true`) to validate before going live.
- **API key security:** When creating the Gemini API key, disable **Withdrawals**. Only enable Balances and Orders.
- **Sideloading:** A personal Apple developer certificate allows sideloading to a limited number of devices for development. Renew/provisioning profiles expire; re-sign via esign as needed.
- **Not financial advice:** This is a technical build prompt, not a recommendation to trade.
