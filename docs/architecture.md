# Marketman Architecture: 5-State Machine Design

## Overview

Marketman is a **deterministic state machine** with five sequential states that map institutional market behavior into actionable entry signals. Each state has discrete conditions and thresholds that must be met before progression.

---

## State 1: Liquidity Sweep Detection

**Purpose**: Identify institutional stop-hunts at structural extremes (VAL/VAH, PDL/PDH).

### Conditions

A sweep occurs when:
1. **Price penetrates** VAL (bullish sweep) or VAH (bearish sweep) by `sweep_points`
2. **Volume spike** exceeds `sweep_volume_multiplier × 20-bar average`
3. **Delta confirms direction** (cumulative delta rises for bullish, falls for bearish)
4. **Price returns** above VAL (bullish) or below VAH (bearish) in same bar or next

### Logic (Pine Script)

```pine
bool sweepLong = low < sessionVAL and close > sessionVAL and deltaBullish
bool sweepShort = high > sessionVAH and close < sessionVAH and deltaBearish
```

### Thresholds

| Contract | Sweep Points | Volume Multiplier |
|----------|--------------|-------------------|
| MNQ      | 6            | 1.3x              |
| MES      | 1.5          | 1.2x              |

### State Transition

→ **State 2** if sweep confirmed + subsequent candles show momentum rejection

---

## State 2: Break of Structure (BOS) / Change of Character (CHoCH)

**Purpose**: Confirm that the sweep was real displacement, not a false liquidity hunt.

### Conditions

BOS is confirmed when:
1. **Price does NOT retrace** back into the VAL/VAH (no immediate trap reversal)
2. **Momentum sustained** for 2–3 bars in entry direction
3. **Delta remains positive** (bullish BOS) or negative (bearish BOS)
4. **New impulse high/low** establishes above the sweep high or below sweep low

### Logic (Pine Script)

```pine
// After sweepLong detected in State 1
bool bosLong = close > high[1] and deltaBullish and ta.rising(delta, 2)
```

### Thresholds

| Contract | BOS Points | Required Bars |
|----------|-----------|---------------|
| MNQ      | 5         | 2–3           |
| MES      | 1.25      | 2–3           |

### State Transition

→ **State 3** if BOS confirmed (no immediate reversal into sweep)

---

## State 3: Retracement into Value Zone

**Purpose**: Identify controlled pullbacks where institutional buyers/sellers absorb and then hold.

### Conditions

A valid retracement occurs when:
1. **Price pulls back** from BOS high/low by 25–65% of impulse move
2. **Price lands in a value zone**:
   - FVG (bullish: between two bearish candles; bearish: between two bullish candles)
   - VWAP ±1σ range
   - EMA 20/50 value area
   - Order block zone
   - Consolidation band
3. **Volume diminishes** relative to sweep bar (absorption phase)
4. **Tight range** formed (range width < `retrace_tight_range_percent × ATR`)

### Logic (Pine Script)

```pine
bool retraceLong = 
    low >= sessionVAL - tolATR and 
    close <= sessionVWAP + dev and 
    volume < ta.sma(volume, 20) * 1.1 and
    (high - low) < atr14 * retrace_tight_range_percent

bool valueZoneLong = 
    priceAtVAL or 
    priceAtVWAPLower or 
    priceInFVG or 
    priceInOrderBlock
```

### Thresholds

| Contract | Retrace Tight Range | VWAP Sigma |
|----------|-------------------|-----------|
| MNQ      | 0.3%              | 1.25–1.5  |
| MES      | 0.25%             | 1.0–1.25  |

### State Transition

→ **State 4** if retracement completes AND one of the three pillars triggers

---

## State 4: 3-Pillar Confirmation

**Purpose**: Validate the retracement is institutional absorption, not a trap.

### The Three Pillars

#### Pillar 1: Location (Price in Value Zone)

Price must be within one of:
- **FVG**: Fair Value Gap (unfilled imbalance)
- **VWAP ±1σ**: Session VWAP value band
- **EMA 20/50**: Moving average support/resistance
- **Order Block**: Prior impulse zone
- **POC/VAH/VAL**: Volume profile levels

```pine
bool locationOK = 
    priceAtVAL or 
    priceAtVWAPLower or 
    priceInFVG or 
    priceInOrderBlock or
    priceAtPOC
```

#### Pillar 2: Rejection (Candle Pattern)

Price must reject lows/highs with one of:
- **Rejection Wick**: Low/high with close away from extreme (wick > body)
- **Engulfing**: Current candle body engulfs prior candle
- **Micro-BOS**: Brief penetration + immediate retrace (1–2 bar pattern)

```pine
bool rejectionWick = (high - close) > (close - open) * 2  // bullish
bool engulfing = close > open[1] and open < close[1]      // bullish
bool microBOS = low < low[1] and close > high[1]          // bullish
```

#### Pillar 3: Participation (Volume/Momentum)

One of:
- **Volume Spike**: Volume ≥ 20-bar average
- **VWAP Reclaim**: Price crosses and holds above/below VWAP
- **EMA Hold**: EMA 20 slopes in entry direction while price stays above it

```pine
bool volumeParticipation = volume >= ta.sma(volume, 20)
bool vwapParticipation = close > sessionVWAP and deltaBullish
bool emaParticipation = close > ema20 and ta.rising(ema20, 2)
```

### Confirmation Logic

**All three pillars must be TRUE:**

```pine
bool confirmation = locationOK and rejectionOK and participationOK
```

### State Transition

→ **State 5** if all three pillars pass on confirmation candle close

---

## State 5: Continuation Breakout → Entry Trigger

**Purpose**: Execute entry on momentum confirmation from a validated retracement.

### Entry Conditions

Entry triggers when:
1. **Confirmation candle closes** with all three pillars met
2. **Next bar (or same bar momentum)** breaks high of confirmation candle + `bos_points`
3. **Volume increases** above 20-bar average
4. **Session phase allows** (not lunch 11:30–13:30 ET, prefer 9:30–10:00 or 14:30–16:00)
5. **Risk can be calculated** and position size ≥ minimum lot

### Entry Price

Entry is triggered at:
- **Market on bar close** if confirmation candle met all pillars
- **Stop entry above** confirmation high + 1 tick if scalping

### Stop Placement

**Long Entry**:
- Primary: Below sweep level (State 1 low)
- Secondary: Below retracement structure (State 3 low)
- Tight: 1.1 × ATR below entry (for tight trails)

**Short Entry**:
- Primary: Above sweep level (State 1 high)
- Secondary: Above retracement structure (State 3 high)
- Tight: 1.1 × ATR above entry

```pine
float longStop = math.max(
    sweepLow - stopBuffer,
    retraceLow - stopBuffer,
    close - atr14 * 1.1
)
```

### Position Sizing

```python
risk_amount = account_balance × 0.01  # 1% per trade
stop_distance = abs(entry - stop)
position_size = risk_amount / (stop_distance × tick_value)
position_size = max(min_contracts, floor(position_size))
```

### Profit Targets

**TP1**: 1.75–2.0R (take partial profit)
- Size: 50% of position
- Price: `entry + stop_distance × 1.75`

**TP2**: Oppose liquidity or 2.5–3.0R
- Size: 25% of position

**Runner**: Trail via EMA 20 or 5-minute swing high/low
- Size: 25% of position
- Exit: When price closes below EMA20 or breaks 5M support

### Entry Logic (Pine Script)

```pine
bool entryReady = 
    inSession and 
    confirmation and 
    volume >= ta.sma(volume, 20) and 
    not inLunch and
    (openPhase or pmPhase)

if entryReady and strategy.position_size == 0
    strategy.entry("Long", strategy.long, qty=positionSize)
    strategy.exit("TP1", from_entry="Long", limit=tp1Price, qty=qty_half)
    strategy.exit("TP2", from_entry="Long", limit=tp2Price, qty=qty_quarter)
    strategy.exit("Stop", from_entry="Long", stop=stopPrice)
```

### State Transition

→ **Exit Logic** (not a new state):
- **DOM Flip**: If buy aggression flips to sell for 2+ bars while long
- **Delta Divergence**: New high but delta lower (exhaustion)
- **Lunch Exit**: Force close at 13:30 ET
- **TP Targets Hit**: Execute partial and runner logic

---

## Summary: State Flow Diagram

```
[Sweep Detected]
    ↓
[BOS/CHoCH Confirmed]
    ↓
[Retracement into Value]
    ↓
[3-Pillar Confirmation Met]
    ↓
[Continuation Breakout]
    ↓
[ENTRY]
    ├→ [TP1 Hit] (50% off, runner continues)
    ├→ [TP2 Hit] (25% off)
    ├→ [Runner Exit] (25% via EMA20)
    └→ [Stop Hit or DOM Flip or Lunch Close]
```

---

## Key Design Principles

1. **No Discretion**: All conditions are binary yes/no. No judgment calls.
2. **State Sequence**: States must occur in order; no skipping.
3. **Contract Awareness**: MNQ and MES use different thresholds.
4. **Risk First**: Entry size and stop are calculated before entry; no exceptions.
5. **Prop-Firm Safe**: Risk per trade, daily loss limit, weekly drawdown cap enforced.
6. **Webhook Ready**: Entry/exit alerts are JSON formatted for automation.
7. **One Threshold Change**: Backtest changes one at a time; never bulk parameter sweeps.

---

## Implementation Checklist

- [ ] Session detection (NY 9:30–16:00 ET)
- [ ] Volume profile (POC/VAH/VAL calculation)
- [ ] VWAP + bands (session and deviation)
- [ ] EMA 20/50 + trend context
- [ ] RSI 14 (adaptive thresholds for volatility)
- [ ] ATR 14 (stop sizing and tight range detection)
- [ ] Cumulative delta (direction + exhaustion)
- [ ] FVG detection (bullish + bearish)
- [ ] Order block zones
- [ ] DOM bias (buy/sell aggression + flip detection)
- [ ] State machine logic (States 1–5)
- [ ] Position sizing formula
- [ ] Webhook JSON output
- [ ] Backtest validation (MNQ + MES, multiple regimes)

