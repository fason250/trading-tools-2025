# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Repository Overview

This is a TradingView Pine Script indicator for Multi-Timeframe Smart Money Concepts (MTF-SMC) trading. The repository contains a complete trading system with documentation and reference materials, designed for trading XAUUSD (Gold) with institutional trading principles.

## Project Structure

- **mtf-smc-jeyfason.pine** (1126 lines) - Main TradingView indicator implementing SMC trading logic
- **reference-tools/** - Reference Pine Script implementations (mxwill-capital.pine, smc-luxalgo.pine)
- **Documentation files:**
  - README.md - Comprehensive user guide with trading rules and examples
  - START-HERE.md - Navigation and onboarding guide
  - QUICK-START.md - Fast reference for daily trading
  - TRADING-CHECKLIST.md - Pre-trade, during-trade, and post-trade checklists
  - WHAT-I-BUILT-FOR-YOU.md - Architectural explanation and design rationale

## Development Commands

### Working with Pine Script

**Testing the indicator:**
1. Copy the code from `mtf-smc-jeyfason.pine`
2. Open TradingView → Pine Editor
3. Paste code → Click "Add to Chart"
4. Test on XAUUSD chart with 1H or 15min timeframe

**No build system** - Pine Script is interpreted directly by TradingView platform

**Version control:**
```bash
# View changes
git diff mtf-smc-jeyfason.pine

# Commit changes
git add mtf-smc-jeyfason.pine
git commit -m "Brief description of changes"
```

## Code Architecture

### High-Level Design Philosophy

The indicator implements a **two-tier signal generation system**:

1. **Pro Sniper Mode** - High-quality setups requiring full confluence:
   - Extreme unmitigated order blocks
   - Liquidity sweeps
   - Displacement (FVG detection)
   - Multi-timeframe bias alignment
   - Structure breaks (BOS/CHoCH)
   - 3 take-profit levels (1:2, 1:3, 1:5 R:R)

2. **Quick Setup Mode** - Zone reaction entries:
   - Engulfing or rejection patterns
   - Premium/Discount zone boundaries
   - Optional order block or FVG confluence
   - 2 take-profit levels (1:1.5, 1:2 R:R)
   - Daily limits and cooldown periods

### Core Data Structures

**OrderBlock type** - Represents institutional entry zones:
- `top/bottom`: Price levels defining the OB
- `isBullish`: Direction of the block
- `mitigated`: Whether price has revisited this zone
- `isExtreme`: Unmitigated blocks from major swings
- `display`: Box object for visualization

**TradeSetup type** - Manages active trade projections:
- Entry, SL, and multiple TP levels
- Visual elements (lines, boxes, labels)
- State tracking (`active`, `movedToBE`)
- Setup classification (`isQuick`, `setupType`, `quality`)
- Win probability estimation

**Structure type** - Tracks swing highs/lows:
- `level`: Price of the swing point
- `barTime/barIndex`: When the swing formed
- `crossed`: Whether price has broken this level (BOS detection)
- `label`: LL/HL (bullish) or HH/LH (bearish)

### Multi-Timeframe Logic

**Bias Calculation** (lines 321-352):
- Fetches 3 HTF candles: 1H (HTF1), 4H (HTF2), Daily (HTF3)
- Three bias modes configurable:
  - "Strict (3/3)": All 3 must align
  - "Majority (2/3) + Daily Anchor": Daily must align + 2/3 total
  - "Daily Only": Only Daily direction matters
- Weekly flip cooldown prevents whipsaw signals

**Premium/Discount Zones** (lines 216-223):
- Calculated from HTF high/low range
- DISCOUNT: 0-38.2% of range (Fibonacci)
- PREMIUM: 61.8-100% of range
- EQUILIBRIUM: Middle zone (38.2-61.8%)

### Structure Detection Algorithm

**Swing Point Detection** (lines 354-400):
- Uses `getLeg()` function with configurable `swingLength` (default 25)
- Detects when price breaks highest high or lowest low over lookback period
- On leg change:
  - Creates new swing high/low Structure
  - Labels as HH/LH (bearish) or LL/HL (bullish)
  - Triggers order block creation
  - Resets per-leg counters

**Break of Structure (BOS)** (lines 404-445):
- Bullish BOS: Price closes above previous swing high
- Bearish BOS: Price closes below previous swing low
- Updates `trend` variable (1=bullish, -1=bearish, 0=neutral)
- Triggers CHoCH (Change of Character) if trend reverses

### Signal Generation Flow

**Pro Sniper Entry Requirements** (lines 516-661):
1. No active setup exists
2. Structure break confirmed (BOS)
3. HTF bias aligned with direction
4. In correct zone (discount for buy, premium for sell)
5. At extreme order block OR swept liquidity
6. FVG displacement present (optional)
7. Cooldown: Max 1-2 entries per leg
8. Win probability calculation based on confluence

**Quick Setup Entry Requirements** (lines 759-925):
1. No active setup exists
2. In correct zone boundary (<15% from edge by default)
3. Engulfing OR rejection pattern with liquidity sweep
4. Body size > 0.6x ATR
5. Matching order block presence (optional)
6. Daily limit: Max 3 quick trades per day
7. Cooldown: 10 bars between signals

### Trade Management System

**Break-Even Logic** (lines 975-1012):
- For Sniper: Moves SL to entry when TP1 is hit
- For Quick: Moves SL to entry when TP1 is hit
- Implemented via `movedToBE` flag to prevent multiple moves
- Automatically updates SL line visualization

**Win Rate Tracking** (lines 975-1003):
- Counts setups that hit TP2 as wins
- For Sniper: Also counts partial wins at TP1
- Updates `wonSetups` and `totalSetups` counters
- Displays percentage in dashboard

**Visual Projection System** (lines 932-973):
- Extends lines and boxes dynamically to current bar
- Risk box (entry to SL) in orange
- Reward boxes (entry to TPs) in green gradients
- Live progress box shows current P&L in real-time
- Color changes: green when profitable, red when in drawdown

### Dashboard Components (lines 1046-1116)

Displays in table format at top-right:
1. **Zone**: Current position (Discount/Premium/Equilibrium)
2. **Trend**: Current structure direction
3. **HTF Bias**: Multi-timeframe alignment status
4. **Structure**: BOS confirmation status
5. **Order Block**: Whether price is in an OB
6. **Quality**: Setup quality rating (HIGH/MEDIUM/LOW)
7. **Setup Mode**: Active setup type or readiness status
8. **Win Rate**: Historical performance percentage
9. **Break-Even**: Alert when TP1 hit and should move SL

## Key Configuration Parameters

### Structure Settings
- `swingLength` (default 25): Higher = only major swings, reduces noise
- `biasMode`: Controls HTF alignment strictness
- `weeklyBlockBars` (default 0): Blocks signals after weekly reversal

### Risk/Reward Settings
- **Sniper**: rr1=2.0, rr2=3.0, rr3=5.0
- **Quick**: quickRR1=1.5, quickRR2=2.0

### Quick Setup Filters
- `minWickRatio` (0.6): Rejection wick must be 60% of candle
- `quickMinBodyATR` (0.6): Minimum candle body size
- `quickBoundaryPct` (0.15): Max 15% distance from zone edge
- `quickCooldownBars` (10): Bars between quick signals
- `quickMaxPerDay` (3): Daily limit for quick setups

## Common Modifications

### Adjusting Signal Frequency

**To get more signals:**
- Reduce `swingLength` to 15-20 (more swing points)
- Change `biasMode` to "Daily Only" (less strict HTF requirement)
- Increase `quickMaxPerDay` to 5-10
- Set `enableProSniper` to false (only use Quick setups)

**To get fewer, higher-quality signals:**
- Increase `swingLength` to 30-40 (only major swings)
- Change `biasMode` to "Strict (3/3)"
- Set `enableQuick` to false (only use Sniper mode)
- Add `weeklyBlockBars` = 5-10 (block after weekly flip)

### Changing Risk/Reward Profile

**More aggressive (tighter stops, bigger wins):**
- Increase `rr2` and `rr3` to 4.0 and 7.0
- SL placement uses structure, but can modify OB mitigation method

**More conservative (wider stops, safer targets):**
- Reduce `rr1` and `rr2` to 1.5 and 2.0
- Change `obMitigation` to "High/Low" instead of "Close"

### Adding New Setup Types

To add a new signal type:
1. Define conditions in main logic section (after line 750)
2. Create TradeSetup object with entry/SL/TP levels
3. Add visual elements (lines, boxes, labels)
4. Update `currentSetup` variable
5. Set appropriate `setupType` and `quality` fields
6. Add alertcondition at end of script

## Important Implementation Details

### Order Block Creation
- Bullish OB: Last red candle before swing low (institutions sold, creating the low)
- Bearish OB: Last green candle before swing high (institutions bought, creating the high)
- Max OBs shown controlled by `maxOB` parameter
- Auto-deletion when mitigated (price revisits)

### FVG Detection (Lines 160-189)
- Bullish FVG: Current low > high from 2 bars ago (gap up)
- Bearish FVG: Current high < low from 2 bars ago (gap down)
- Represents unfilled orders/imbalance
- Used for displacement confirmation in Sniper setups

### Cooldown Mechanisms
- **Pro Sniper**: `proEntriesThisLeg` limits entries per leg (1-2 max)
- **Quick Setup**: `lastQuickBar` enforces bar-based cooldown
- **Daily Reset**: `quickCountToday` resets when day changes

### Request.security Considerations
- Uses `lookahead=barmerge.lookahead_on` for HTF data
- Fetches [1] bars to avoid repainting
- All HTF requests use same security() call for efficiency

## Testing Workflow

1. **Backtesting**: Load historical data on TradingView, scroll through chart
2. **Strategy Tester**: Convert to `strategy()` instead of `indicator()` for automated backtesting
3. **Paper Trading**: Use alerts + manual execution before live trading
4. **Live Monitoring**: Dashboard provides real-time status without chart watching

## Alert Configuration

Available alert types:
- `🚀 ENTRY SIGNAL` - New setup triggered
- `🎯 MOVE TO BREAK-EVEN` - TP1 hit, move SL
- `🟢 DISCOUNT Zone` - Price entered buy zone
- `🔴 PREMIUM Zone` - Price entered sell zone
- `📈 Bullish BOS` - Bullish structure break

Set alerts to "Once Per Bar Close" to avoid false triggers.

## Documentation Maintenance

When modifying code logic:
1. Update inline comments in Pine Script
2. Update relevant sections in README.md (user-facing features)
3. Update QUICK-START.md if default settings change
4. Update TRADING-CHECKLIST.md if trading rules change
5. Keep WHAT-I-BUILT-FOR-YOU.md aligned with core architecture

## Performance Considerations

- Limit max_boxes/lines/labels to 500 (declared at top)
- Array cleanup: Old OBs auto-deleted when exceeding `maxOB`
- Visual cleanup: Delete all visual elements when trade completes
- State management: Use `var` for persistent variables across bars

## Common Troubleshooting

**No signals appearing:**
- Check HTF bias alignment (most common issue)
- Verify structure detection with lower `swingLength`
- Ensure in correct zone (discount for buys, premium for sells)

**Too many signals:**
- Increase `swingLength` for cleaner structure
- Disable Quick setups (`enableQuick = false`)
- Switch to "Strict (3/3)" bias mode

**Signals not matching expected behavior:**
- Review current dashboard values for context
- Check if weekly flip cooldown is active
- Verify order blocks are showing on chart
- Confirm current zone calculation is correct
