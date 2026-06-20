---
applyTo:
  - "**/*.pine"
  - "**/*.py"
  - "**/docs/**"
  - "**/config/**"
  - "**/webhook/**"
description: "Marketman state-machine strategy rules for Copilot agents. Use this when developing, debugging, or extending the 5-state trading engine."
---

# Marketman Strategy Instructions for Copilot Agents

## Core Principle

Marketman is a **deterministic state machine with contract-specific thresholds**. There is no discretion, no judgment calls. Every condition is binary: triggered or not.

## When These Instructions Apply

Use these rules when:
- Writing or debugging the 5-state machine logic
- Adding new indicators or refining existing ones
- Tuning thresholds for MNQ or MES
- Implementing entry/exit logic
- Validating webhook alert JSON output
- Extending position sizing or risk management

## The 5 States

Do not skip states. They must occur in order:

1. **Sweep**: Institutional liquidity hunt at VAL/VAH or PDL/PDH
2. **BOS/CHoCH**: Confirmation that displacement is real, not a trap
3. **Retracement**: Controlled pullback into a value zone (FVG, VWAP ±σ, etc.)
4. **Confirmation**: 3-pillar validation (Location + Rejection + Participation)
5. **Entry**: Continuation breakout with all green lights

## Contract-Specific Thresholds

**Always reference the correct thresholds.** They are NOT interchangeable.

### MNQ (Nasdaq-100 Micro)
- Sweep: 6 points, volume multiplier 1.3x
- BOS: 5 points, 2–3 bars confirmation
- Retrace tight range: 0.3% (30 bps)
- VWAP sigma: 1.25–1.5
- RSI normal: 70/30 bounds; low vol: 60/40 bounds

### MES (S&P 500 Micro)
- Sweep: 1.5 points, volume multiplier 1.2x
- BOS: 1.25 points, 2–3 bars confirmation
- Retrace tight range: 0.25% (25 bps)
- VWAP sigma: 1.0–1.25
- RSI normal: 70/30 bounds; low vol: 60/40 bounds

See `/config/thresholds.json` for all parameters.

## 3-Pillar Confirmation (State 4)

**All three must pass.** No exceptions.

### Pillar 1: Location
Price must be in one of these value zones:
- FVG (Fair Value Gap)
- VWAP ±1σ band
- EMA 20/50 value area
- Order block zone
- POC/VAH/VAL

### Pillar 2: Rejection
Price must reject the retracement low/high with one of:
- Rejection wick (wick > body size)
- Engulfing candle (current body engulfs prior)
- Micro-BOS (brief penetration + retrace in 1–2 bars)

### Pillar 3: Participation
One of:
- Volume ≥ 20-bar average
- VWAP reclaim (price crosses and holds above/below VWAP + bullish delta)
- EMA20 hold (price stays above EMA20 while it slopes up in long entry)

## Entry Rules

- **No entries during lunch**: 11:30–13:30 ET (enforced)
- **Preferred entry phases**: 9:30–10:00 (open) or 14:30–16:00 (PM trend)
- **Position size**: Calculated before entry; never discretionary
- **Stop placement**: Below sweep (State 1) or retracement (State 3) structure
- **No skipping TP1**: Always take at least 50% off at 1.75–2.0R

## Exit Logic (Not a State)

Exits are **not states**. They can trigger from any position:

- **TP Targets**: Partial exits at TP1 (50%) and TP2 (25%); runner trails EMA20 or 5M swing
- **DOM Flip**: 2+ consecutive bars of opposing DOM bias
- **Delta Divergence**: New high but delta lower (exhaustion), or vice versa
- **Lunch Close**: Force close all positions at 13:30 ET
- **Stop Hit**: Stop loss breach (hard exit)

## Risk Management (Lucid Trading Rules)

**Enforce these without exception:**

```
risk_amount = account_balance × 0.01
position_size = risk_amount / (stop_distance × tick_value)
```

- **Per-trade risk**: 1% max
- **Daily loss limit**: –2% (stop trading day after this)
- **Max 3 losses per day**: If third loss hits, reduce size by 50% or stop
- **Weekly drawdown cap**: –5% (auto-reduce size)
- **Min position size**: 1 contract (no fractional shares in futures)

## Webhook Output

All alerts must be JSON-formatted and match the schema in `/webhook/schema.json`.

### Entry Alert (Minimum Required Fields)
```json
{
  "event": "entry_trigger",
  "symbol": "MNQ",
  "direction": "LONG",
  "entry_price": 30700.50,
  "stop_loss": 30694.00,
  "tp1_price": 30710.50,
  "tp1_size": 0.5,
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

### Exit Alert (Minimum Required Fields)
```json
{
  "event": "exit_trigger",
  "symbol": "MNQ",
  "position_direction": "LONG",
  "exit_price": 30710.50,
  "entry_price": 30700.50,
  "exit_reason": "tp1_hit",
  "profit_loss_usd": 105,
  "contracts_closed": 1
}
```

## Tuning and Backtesting

### Golden Rules
1. **One threshold at a time**: Never bulk parameter sweep
2. **Backtest before production**: Always validate on 1-minute data, multiple market regimes
3. **Test both contracts**: MNQ and MES have different volatility; thresholds may need adjustment
4. **Avoid overfitting**: If a threshold only works on specific dates, it's overfitted
5. **Document changes**: Record why a threshold was changed and what problem it solved

### Validation Checklist
- [ ] All 5 states trigger in correct order
- [ ] Sweeps filter false breakouts (volume + delta confirmation)
- [ ] BOS holds for 2–3 bars (no immediate reversals)
- [ ] Retracements stay in value zones (no gaps into thin air)
- [ ] All 3 pillars pass before entry (no partial confirms)
- [ ] Entry size calculated correctly per risk formula
- [ ] Stop placed below/above structure, not arbitrary
- [ ] TP1 at 1.75–2.0R (proportional to stop distance)
- [ ] Lunch exits enforced (11:30–13:30 ET)
- [ ] Daily loss limit prevents over-trading after drawdown
- [ ] Webhook JSON matches schema

## Code Style for Pine Script

```pine
// Threshold inputs at top (always MNQ/MES specific)
float sweep_points_mnq = input.float(6.0, "Sweep Points (MNQ)")
float sweep_points_mes = input.float(1.5, "Sweep Points (MES)")

// State detection functions (one per state)
f_detect_sweep(priceLevel, sweepPoints, deltaConfirmed) =>
    // Logic here
    result

// Entry/exit logic (clean, readable, no magic numbers without comment)
bool entryReady = state4_confirmation and state5_breakout
if entryReady and strategy.position_size == 0
    strategy.entry("Long", strategy.long, qty=positionSize)
    alert('{"event":"entry_trigger", ...}')  // Webhook output

// Comments for complex logic
// ponytail: Using simple volume > SMA check; could upgrade to VWAP volume profile if needed
bool volumeOK = volume >= ta.sma(volume, 20)
```

## Python Backtester Considerations

If using Python (`strategy.py`):

- OHLCV data must be **1-minute bars** (TradingView cannot properly backtest sub-minute session logic)
- Session detection: Filter to NY hours only (9:30–16:00 ET)
- Contract detection: Auto-detect MNQ vs MES from symbol string
- Thresholds: Load from `/config/thresholds.json` or hardcode with clear comments
- Risk calculation: Must match the formula exactly (no approximations)
- Webhook output: Generate JSON strings for each entry/exit (for later forwarding to order router)

## Common Mistakes to Avoid

1. **Skipping states**: Don't enter on sweep alone or on retracement without confirmation
2. **Wrong thresholds**: Using MNQ thresholds for MES (or vice versa)
3. **Partial pillar confirms**: Entry requires ALL 3 pillars; 2/3 is not enough
4. **Disclosing stop placement**: Use structure, not arbitrary ATR distance
5. **Ignoring lunch**: Never enter 11:30–13:30 ET
6. **Overfitting thresholds**: Tuning for one specific day will fail on another
7. **Forgetting risk scaling**: Position size must be calculated; no discretion
8. **Skipping TP1**: Always take at least 50% off at first target
9. **Missing webhook schema**: Alerts must match JSON schema exactly
10. **Not documenting changes**: Every threshold change must be logged with reason

## Examples

### Example 1: MNQ Long Entry
```
Sweep detected: Price breaks VAL by 6 points, volume 1.3x average, delta positive ✓
BOS confirmed: Price doesn't retrace; delta sustains for 3 bars ✓
Retrace complete: Price pulls back to VWAP ±1σ in 0.3% tight range ✓
Confirmation: Location=VWAP_SIGMA, Rejection=ENGULFING, Participation=VOLUME_SPIKE ✓
Entry: Break of confirmation candle at 9:45 AM (open phase) ✓
Position: 1 contract (risk $100, stop $6 away = 0.5/6 × 1 = 0.08c = round to 1)
Alert: JSON webhook sent to order router ✓
```

### Example 2: MES Short Exit (TP1)
```
Entry: 30-min-old short at 4120.25
Market: Price drops to 4118.00 (2.25R from entry)
TP1: 4119.50 (1.75R) hits; 50% position exits
Alert: {"event":"exit_trigger", "exit_reason":"tp1_hit", "profit_loss_usd": 275}
Runner: 50% trails EMA20; if EMA breaks, close runner
```

## See Also

- [`/docs/architecture.md`](/docs/architecture.md) — Detailed 5-state system design
- [`/config/thresholds.json`](/config/thresholds.json) — All contract thresholds
- [`/webhook/schema.json`](/webhook/schema.json) — Alert JSON schema
- [`.github/instructions/triad-trading.instructions.md`](.github/instructions/triad-trading.instructions.md) — Triad reasoning workflow
- [`.github/copilot-instructions.md`](.github/copilot-instructions.md) — Lazy senior dev philosophy

