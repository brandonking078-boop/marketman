# Marketman: State-Machine-Driven Intraday Futures Trading Engine

A deterministic, prop-firm-safe trading system for **MNQ (Nasdaq-100 Micro)** and **MES (S&P 500 Micro)** built on institutional market structure and order flow analysis.

## 🎯 Core Strategy

Marketman detects and executes a specific market sequence:

```
Liquidity Sweep → Structure Shift (BOS/CHoCH) → Retracement → Confirmation → Continuation Entry
```

### Not an Indicator Mashup
This is a **state machine with contract-specific thresholds** that:
- Detects institutional stop-hunts (liquidity sweeps)
- Confirms real displacement (BOS/CHoCH)
- Identifies controlled pullbacks into value zones
- Validates entries using a 3-pillar confirmation system
- Triggers entries only on continuation (not on sweeps)
- Sizes positions based on evaluation-account risk rules
- Outputs webhook-ready alerts for automation

## 🏗️ System Architecture

### 5-State State Machine

The entire strategy operates through these states:

| State | Description | Trigger |
|-------|---|---|
| **State 1: Sweep** | Institutional liquidity hunt detected | Price breaks VAL/VAH, volume spike, delta confirmation |
| **State 2: BOS/CHoCH** | Real displacement confirmed | Price fails to reclaim sweep level + momentum |
| **State 3: Retracement** | Controlled pullback into value | Price returns to FVG, consolidation, or VWAP ±σ |
| **State 4: Confirmation** | 3-pillar validation passed | Location + Rejection + Participation all met |
| **State 5: Entry** | Continuation breakout trigger | Break of confirmation candle, all conditions green |

**Each state is a separate module** with its own thresholds and logic.

### Contract-Specific Thresholds

**MNQ (Nasdaq-100 Micro)**
- Sweep points: 6
- BOS points: 5
- Sweep volume multiplier: 1.3
- Retrace tight range: 0.3%
- VWAP sigma: 1.25–1.5

**MES (S&P 500 Micro)**
- Sweep points: 1.5
- BOS points: 1.25
- Sweep volume multiplier: 1.2
- Retrace tight range: 0.25%
- VWAP sigma: 1.0–1.25

Thresholds are inputs in Pine Script for live tuning. See [`/config/thresholds.json`](config/thresholds.json).

### 3-Pillar Confirmation System (State 4)

All three pillars must pass before State 5 can trigger:

**Pillar 1 — Location** (Price in valid value zone)
- FVG (Fair Value Gap)
- Consolidation band
- VWAP ±1σ
- EMA 20/50 value area
- Order block

**Pillar 2 — Rejection** (Price rejects lows/highs)
- Rejection wick
- Engulfing candle
- Micro-BOS

**Pillar 3 — Participation** (Volume/momentum confirmation)
- Volume ≥ 20-bar average
- VWAP reclaim/loss
- EMA20/EMA50 hold

### Entry Logic (State 5)

- Entry: Break of confirmation candle
- Stop: Below/above sweep or retracement structure
- TP1: 1.75–2R or opposing liquidity
- Runner: Trailing via EMA20 or 5M swing structure

### Evaluation-Account Risk Model (Lucid Trading)

- **Per-trade risk**: 1% account equity
- **Daily loss limit**: –2%
- **Max 3 losses per day**
- **Weekly drawdown cap**: –5%
- **Auto-reduce size after drawdown**

**Position Sizing Formula:**
```
risk_amount = account_balance × risk_percent
stop_distance = |entry − stop|
contracts = risk_amount / (stop_distance × tick_value)
```

## 📁 Repository Structure

```
marketman/
├── README.md (this file)
├── LICENSE
├── config/
│   └── thresholds.json          # MNQ/MES threshold inputs
├── docs/
│   └── architecture.md          # Detailed 5-state system design
├── webhook/
│   └── schema.json              # Entry/exit alert JSON format
├── .github/
│   ├── copilot-instructions.md  # Lazy senior dev philosophy
│   ├── instructions/
│   │   ├── triad-trading.instructions.md       # Triad reasoning workflow
│   │   └── marketman-strategy.instructions.md  # Marketman-specific agent rules
│   └── agents/
│       ├── trading-strategy.agent.md           # Strategy agent
│       └── marketman.agent.md                  # Marketman specialist agent
├── marketman_intraday_profile_strategy.pine    # v1 strategy (DOM/order flow)
├── marketman_intraday_profile_strategy_v2.pine # v2 strategy (phase-aware)
├── marketman_intraday_profile_indicator.pine   # Indicator overlay for TradingView
└── strategy.py                                  # Python backtest utilities
```

## 🔌 Webhook Output Format

Alerts are JSON-formatted for automation and order routing:

```json
{
  "event": "entry_trigger",
  "symbol": "MNQ",
  "direction": "LONG",
  "entry_price": 30700.50,
  "stop_loss": 30694.00,
  "tp1_price": 30710.50,
  "tp1_size": 0.5,
  "runner_trail": "ema20",
  "position_size": 2,
  "risk_usd": 100,
  "state": "continuation_breakout",
  "confirmation_pillars": {
    "location": "vwap_sigma",
    "rejection": "engulfing",
    "participation": "volume_spike"
  }
}
```

See [`/webhook/schema.json`](webhook/schema.json) for full schema.

## 🧠 For Copilot Agents

This repository is instrumented for intelligent agent collaboration:

- **Triad Reasoning**: All strategy decisions go through devil's advocate, professional analyst, and emotion analyst lenses.
- **Modular State Logic**: Build/debug each state independently, then integrate.
- **Contract-Specific Thresholds**: Always reference MNQ vs MES inputs before coding.
- **Risk-First**: No entry without position sizing and stop validation.
- **No Overfitting**: One threshold change at a time; backtest before production.

See [`.github/instructions/marketman-strategy.instructions.md`](.github/instructions/marketman-strategy.instructions.md) for detailed agent rules.

## 🚀 Quick Start

1. **TradingView**: Load `marketman_intraday_profile_strategy_v2.pine` into TradingView with MNQ or MES chart.
2. **Backtesting**: Use `strategy.py` utilities with your OHLCV data source.
3. **Webhook Automation**: Set TradingView alerts to POST webhook JSON to your execution adapter (Tradovate, TradeLocker, etc.).

## 📊 Key Indicators

- **Session VWAP + Bands**: Value area anchors
- **POC/VAH/VAL**: Volume profile structure
- **EMA 8/21/50**: Trend context and pullback zones
- **RSI 14**: Momentum boundaries (adaptive for volatility)
- **ATR 14**: Risk scaling and stop placement
- **Cumulative Delta**: Order flow direction + extremes
- **DOM Bias**: Buy/sell aggression tracking

## 🔍 Testing & Validation

- Run backtests on 1M timeframe (TradingView can't backtest < 1M sub-session splits properly).
- Validate across multiple market regimes (trending, range-bound, volatile opens).
- Use the "Lazy Senior Dev" philosophy: one change, one backtest, one threshold at a time.

## 📝 License

See [`LICENSE`](LICENSE) for terms.

---

**Built for Lucid Trading evaluation accounts. Not financial advice.**
