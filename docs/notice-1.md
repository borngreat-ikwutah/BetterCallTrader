# BetterCallTrader — Business Logic Notice #1

Extracted from the codebase (`py/research_bot.py`, `bettercalltrader/exits.py`,
`bettercalltrader/strategies.py`, `bettercalltrader/risk_analytics.py`,
`bettercalltrader/costs.py`, `bettercalltrader/ohlcv.py`,
`bettercalltrader/defaults.py`).

## 1. Discovery

**DiscoveryAgent** (`py/research_bot.py:281`)
- Pulls latest Solana candidates from DexScreener `token-profiles/latest/v1` and
  `token-boosts/latest/v1` (`chainId == "solana"`), dedupes, returns up to
  `limit` (default 5).
- Fallback list: Cupsey (`suqh5sHtr8HyJ7q8scBimULPkPpA557prMG47xCHQfK`), Gake
  (`DNfuF1L62WWyW3pNakVkyGGFzVVhj4Yr52jSmdTyeBHm`), Raydium Whale Watcher
  (`5Q544fKrFoe6tsEbD7S8EmxGTJYAKtTVhAW5Q5pge4j1`).

**PumpFunStreamAgent** (`py/research_bot.py:329`)
- WebSocket `wss://pumpportal.fun/api/data`; subscribes `subscribeNewToken`,
  `subscribeMigration`, `subscribeAccountTrade` (tracked wallets).
- Rejects `txType == "create"` when real SOL in curve > `max_dev_buy_sol`
  (15.0).
- Accepts `create` when `vSolInCurve >= min_initial_sol_in_curve` (30.2);
  `bonding_curve_pct = min(100, (real_sol / 85) * 100)`.
- Records whale sells (`record_whale_sell`, used by WHALE_EXIT).
- Emits tracked-wallet buys as `WHALE_COPY_SIGNAL` candidates.

## 2. Research

**ResearchAgent.get_market_data** (`py/research_bot.py:539`)
- DexScreener `/latest/dex/tokens/{mint}`; picks the best Solana pair by
  liquidity; extracts price, fdv, liquidity, m5/h1 volume, buys/sells, price
  changes, websites, socials.
- Falls back to the Pump.fun bonding-curve quote
  (`frontend-api-v3.pump.fun/coins/{mint}`, 5 s cache, 60 s negative cache);
  price = `usd_market_cap / total_supply` or virtual-reserve ratio.
- Price cache (TTL 3 s).

**get_security_data** (`py/research_bot.py:804`)
- RugCheck full report: `mint_authority_revoked = token.mintAuthority is None`;
  same for freeze authority.
- Top holders: sum of `topHolders[].pct`, skipping the first (>24% — dev/vault),
  capped at 10.
- Bundle detection: risk names contain bundle/sniper/insider, or top-4
  near-symmetric (max−min < 0.25, sum > 9).
- Dev-sold detection (creator + "sold" risk names).
- RPC fallback: `getTokenSupply` + `getTokenLargestAccounts`.

**pass_hard_filters** (`py/research_bot.py:904`) — ordered checks:
- Liquidity ≥ MIN_LIQUIDITY_USD (5000); H1 volume ≥ MIN_VOLUME_H1_USD (2500)
  unless live Pump.fun.
- Fail-closed when security data is missing; top10 ≤ MAX_TOP10_HOLDER_PCT (35).
- Mint & freeze authorities must be revoked; no DANGER risks; not bundled.
- No M5 dump (≤ −8%); sell pressure ≤ buys or ≥ 3 sells; no H1 chase
  (≥ +400% with M5 falling); liquidity/FDV ratio ≥ 5% when FDV ≥ $50k.

**ai_evaluate_token** (`py/research_bot.py:973`)
- Gemini or local OpenAI-compatible LLM via `LLMClient.complete_json`
  (temperature 0, pydantic-parsed `TokenAnalysisDecision`).
- Provider failure → heuristic fallback.

**_heuristic_fallback** (`py/research_bot.py:991`) — risk_score starts 55,
hype_score 40:
- Whale signal +24 hype.
- top10 ≤ 20% → risk+8; > 26% → risk−15; unconfirmed → risk−20 (fail-closed).
- Bundled → risk−22; warn risks → risk −12×n (≤ 30); buy pressure (buys >
  1.3×sells, buys ≥ 8) → hype+22; else sells > buys → hype−15; socials/website
  present → +14 hype, else risk−12.
- Clamped to 5..98. Decision: whale + risk ≥ 72 & hype ≥ 75 → BUY;
  risk ≥ 62 & hype ≥ 58 → WATCH; else SKIP.

## 3. Entry gating & confirmation

**request_entry** (`py/research_bot.py:2684`):
- Blocks: NO_MINT, DUPLICATE, STALE_PRICE (quote older than STALE_PRICE_SEC=15 s;
  future timestamps treated as stale).
- Otherwise PENDING for ENTRY_CONFIRM_SEC=20 s; cancelled if price drops >
  ENTRY_CONFIRM_MAX_DROP_PCT=4% from signal.

**process_confirmation** (`py/research_bot.py:2771`): at due time re-checks —
breaker tripped → CANCELLED; stale → STALE_PRICE; drop > 4% → CONFIRM_DROP;
hard filters re-run at recheck → HARD_FILTER; entry gates → CANCELLED; then
opens the position.

**check_entry_gates** (`py/research_bot.py:2594`): circuit breaker; MAX_OPEN
(open + other pending ≥ max_open_positions, default 2); ENTRY_SPACING (≥60 s);
SL_COOLDOWN (180 s after STOP_LOSS/FAST_INVALIDATION); cluster guard.

**check_cluster_guard** (`py/research_bot.py:2543`): CLUSTER_LOSS_COOLDOWN
(300 s since losing exit); CLUSTER_OPEN_DD (open position > 8% down);
CLUSTER_UNPROVEN (fresh open/pending < 90 s).

## 4. Risk engine (PaperTradingManager)

- **Circuit breaker** (`get_circuit_breaker_state`, `py/research_bot.py:2404`):
  daily limits MAX_DAILY_STOP_LOSSES (2), MAX_DAILY_DRAWDOWN_PCT (5),
  MAX_DAILY_TRADES (8), vs. day-start equity snapshot; resets at local
  midnight; manual reset via `reset_circuit_breaker`.
- **Position sizing** (`compute_position_size`, :2511): base =
  trade_size_sol; ×0.5 after a losing last trade; ×(conviction): high=1.0,
  ai=0.8, heuristic=0.6, manual=1.0; ×0.75 when session down ≥1%; clamp to
  [MIN_POSITION_SOL=0.1, MAX_POSITION_SOL].
- **Wallet quality** (`get_wallet_quality`, :2135): recent win-rate of last 5
  exits; ≥3 samples & ≥40% win rate → trusted, else downweighted; fail-closed
  on errors.
- **Whale-copy trust** (`apply_whale_signal`, :3412): only trusted whales' buys
  set `whale_signal`; their sells flag WHALE_EXIT within
  WHALE_EVENT_TTL_SEC=30 min.

## 5. Exit policy

**ExitPolicy** (`bettercalltrader/exits.py:73`) — one policy shared by the
paper book and the backtester.

Paper order (`classify_tick`, :160): WHALE_EXIT → TAKE_PROFIT_1 (fresh,
pnl ≥ tp1_pct, sells `tp1_fraction`) → FAST_INVALIDATION (age ≤ 90 s, pnl ≤
−12%) → STOP_LOSS (pnl ≤ −25%) → TIME_STOP (age ≥ 900 s, no pump, peak <
+20%) → FLOW_EXIT (sells ≥ 3 and sells > buys, sells ≥ 1.3×buys) → effective
stop → TAKE_PROFIT (final target, only after TP1).

**Effective stop ladder** (`effective_stop`, :115):
- Trailing: arms when highest ≥ entry × (1 + trailing_arm_gain_pct/100) (30%);
  stop = highest × (1 − trailing_stop_pct/100) (20%; 15% after TP1).
- Floors: breakeven = entry × (1 + 1%) after TP1; step-up: entry × (1 + 15%)
  at peak ≥ +50%, entry × (1 − 5%) at peak ≥ +30%.
- Effective stop = max(trail, floor).

Backtester (`intrabar`, :227): gap fills at `min(open, stop)` / `max(open, tp)`;
ambiguous when both stop and target hit the same candle.

## 6. Execution (live)

**SolanaExecutionAgent** (`py/research_bot.py:1068`)
- Jupiter v6 quote + swap for non-Pump tokens; PumpPortal `trade-local` for
  Pump.fun; sign+send via `VersionedTransaction`; Jito block-engine RPC when
  mainnet.
- Staging: `LIVE_TRADING_STAGE` = dry_run → probe → full; probe attempts
  persisted per UTC day (budget `LIVE_STAGE_MAX_TRADES`); requires
  `LIVE_TRADING_CONFIRMED=true`; `MAX_LIVE_TRADE_SOL` 0.05.

## 7. Strategies & profiles

- **momentum** — `EmaCrossStrategy` (`bettercalltrader/ohlcv.py:242`): fast 9 /
  slow 21 EMAs; BUY when fast crosses above slow, SELL when it crosses below.
- **breakout** — `BreakoutStrategy` (:270): lookback 20, threshold 0.5%; BUY
  when close > prior high + 0.5%; SELL when close < prior low − 0.5%.
- **Profiles** (`bettercalltrader/strategies.py`): conservative / balanced
  (default) / aggressive / high_risk (override). High-risk: 1.5 SOL size,
  4 positions, stop −45%, TP1 +250% of 0.35 fraction, trailing 35/25%, arm
  gain 60%, daily limits 5 / 20% / 25.

## 8. Backtester engine (ohlcv.py)

- Event-driven, long-only, one position at a time, fills at candle open
  (latency 1 candle).
- Exits use shared `ExitPolicy.intrabar`; TP1 partial exit (`tp1_fraction` =
  0.5 by default); `max_participation_pct` caps fill by candle volume;
  `close_at_end` → END_OF_DATA.
- Modules: `run_backtest`, `walk_forward` (out-of-sample segments),
  `optimizer` (grid over fast/slow/lookback/threshold; objectives pnl,
  return_pct, profit_factor, expectancy, sharpe; fixed/anchored/rolling
  windows), `cost_stress` (multiplier sweep, break-even), `compare`
  (ranked strategies).

## 9. Cost model (bettercalltrader/costs.py)

- Entry: slippage (base 50 bps + liquidity impact) + AMM fee (25 bps) +
  priority fee 0.0015 + failed-tx expected cost; exit wider (80 bps base) and
  priced for depth consumed at entry (`depth_used_fraction`).

## 10. Risk analytics (bettercalltrader/risk_analytics.py)

- `summarize()`: trades, netPnl, winRate, profitFactor, expectancy,
  maxDrawdownPct, var95, expectedShortfall95, avgWin/Loss; `herfindahl_index`
  for concentration.
