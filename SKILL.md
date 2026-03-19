---
name: skill-smc-multi-strategy-paper-trader
description: Paper trading monitors for SMC (Smart Money Concepts) strategies. Swing (4H BoS+FVG), day (1H + CVD), and coordinated 8D/2S orchestration. ATR-based SL/TP, z-score filters, orchestrator-lock. Binance public API only — no credentials needed.
author: Zero2Ai-hub
version: 1.0.0
emoji: 📈
tags: [crypto, trading, paper-trading, smc, smart-money, orchestrator, backtested, binance]
---

# SMC Multi-Strategy Paper Trader

Paper trading system for crypto using **Smart Money Concepts** across three strategy types. All data from Binance public API — no account or API key required.

## Strategies

| Strategy | Timeframe | Max Hold | Slots | Best Backtest WR |
|----------|-----------|----------|-------|------------------|
| Swing | Daily BoS + 4H FVG | 72h | 5 | ~72% |
| Day (v5) | 1H BoS + FVG + CVD | 12h | 10 | ~75% |
| Coordinated 8D/2S | 1H + 4H mixed | varies | 10 | ~73.6% |

## Architecture

```
Binance Public Kline API
        ↓
paper-monitor-[strategy].js
  ├── Break of Structure (BoS) detection
  ├── Fair Value Gap (FVG) scanner
  ├── Z-score: Volume, TA composite, FVG quality
  ├── ATR-based SL / TP / Trailing stop
  ├── Orchestrator lock (prevents cross-strategy conflicts)
  └── portfolio-[strategy].json (P&L, equity curve)
```

## Orchestrator Lock

All three monitors share `orchestrator-lock.json`. Before entering a trade, each monitor checks if the symbol is already locked by another strategy. On exit (SL/TP/trail/timeout), the lock is released.

```json
{
  "BTCUSDT": { "strategy": "swing", "entryTime": 1742400000000 },
  "ETHUSDT": { "strategy": "day",   "entryTime": 1742405000000 }
}
```

## Signal Logic

### Break of Structure (BoS)
- Bullish BoS: current high breaks above the highest high of last N candles
- Bearish BoS: current low breaks below the lowest low of last N candles
- Only long entries (configurable)

### Fair Value Gap (FVG)
- 3-candle pattern: gap between candle[i-2].high and candle[i].low
- Searched within configurable lookback window after BoS
- Z-score filtered: only high-quality gaps

### ATR-Based Stops
```
SL = entry - (ATR × slAtrMult)
TP = entry + (ATR × tpAtrMult)
Trail: move SL up when price reaches trail trigger
```

### TA Z-Score Composite
Combines 6+ indicators (EMA cross, RSI, MACD, Bollinger, MFI, Stoch).
Minimum score threshold (taMinScore ≥ 5) filters low-conviction setups.

## Usage

```bash
# Install: Node.js only, no npm packages needed
# All uses Node.js built-in https module

# Run swing monitor (call every 4h via cron)
node paper-monitor-swing.js

# Run day monitor with CVD (call every 1h)
node paper-monitor-v5.js

# Run coordinated 8D+2S (call every 1h)
node paper-monitor-coordinated.js
```

## Configuration

Edit the `P` object at the top of each monitor:

```js
const P = {
  maxConcurrent: 5,     // max open positions
  posPct: 0.10,         // 10% of balance per trade
  slAtrMult: 2.5,       // stop loss ATR multiplier
  tpAtrMult: 5.0,       // take profit ATR multiplier
  trailAtrMult: 3.0,    // trailing stop ATR multiplier
  taMinScore: 5,        // minimum TA z-score (0-10)
  maxHold: 72,          // max hold in hours
};
```

## Portfolio File

Each strategy writes its own `portfolio-[strategy].json`:

```json
{
  "balance": 1000,
  "positions": [],
  "history": [],
  "metrics": {
    "totalTrades": 0,
    "wins": 0,
    "losses": 0,
    "winRate": 0,
    "totalPnl": 0
  }
}
```

## Recommended Cron Schedule

```
# Swing: every 4h at :30
30 */4 * * * node /path/to/paper-monitor-swing.js

# Day v5: every hour at :02
2 * * * * node /path/to/paper-monitor-v5.js

# Coordinated: every hour at :05
5 * * * * node /path/to/paper-monitor-coordinated.js
```

## Files

- `paper-monitor-swing.js` — Swing strategy (Daily BoS → 4H FVG)
- `paper-monitor-v5.js` — Day strategy with CVD z-score filter
- `paper-monitor-coordinated.js` — 8D/2S combined orchestrator

## Backtesting

See `backtest-swing-smc.js`, `backtest-cvd.js`, `backtest-coordinated.js` for corresponding backtesting scripts.

Key backtest findings:
- CVD filter: fewer trades but -3.4% max DD vs -15% without
- 8D/2S split: optimal from 65-combination grid search
- Pure TA indicators alone: insufficient edge (z-scoring can't save them)
- FVG + BoS combo: minimum viable signal structure

## Notes

- No external npm dependencies — pure Node.js
- Tested on 29-token watchlist (BTC, ETH, SOL, BNB, XRP, DOGE + 23 alts)
- All paper trades — no real money involved
- ATR uses Wilder's smoothing method
