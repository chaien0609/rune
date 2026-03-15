---
name: "trade-logic"
pack: "@rune/trading"
description: "Trading logic preservation and reasoning — entry/exit spec management, indicator parameter registry, strategy state tracking, and backtest result linkage. Prevents the #1 trading bot failure: AI sessions overwriting working logic without understanding it."
model: sonnet
tools: [Read, Edit, Write, Grep, Glob, Bash]
---

# trade-logic

Trading logic preservation and reasoning — entry/exit spec management, indicator parameter registry, strategy state tracking, and backtest result linkage. Prevents the #1 trading bot failure: AI sessions overwriting working logic without understanding it.

#### Workflow

**Step 1 — Load trading logic context**
Check if `logic-guardian` (L2) has a manifest loaded. If `.rune/logic-manifest.json` exists, read it and extract trading-specific components (ENTRY_LOGIC, EXIT_LOGIC, FILTER, INDICATOR). If no manifest exists, trigger `logic-guardian` Phase 3 to generate one with trading-aware scanning.

Trading-specific file patterns to scan:
- `**/scenarios/**`, `**/signals/**`, `**/strategies/**` — entry/exit logic
- `**/trailing/**`, `**/exit/**`, `**/stoploss/**` — exit engine components
- `**/indicators/**`, `**/core/indicators*` — technical indicator implementations
- `**/backtest/**`, `**/engine*` — backtesting mirrors of production logic
- `**/config/settings*`, `**/config/token*` — parameter source of truth

**Step 2 — Build trading logic spec**
For each trading component, extract a structured spec:

```
COMPONENT: RSI Entry Detector
TYPE: ENTRY_LOGIC
STATUS: ACTIVE (production)
LAYERS: [which layer in the trading pipeline this belongs to]

ENTRY CONDITIONS:
  1. TrendPass ticket exists with available fires
  2. RSI_MA crosses threshold (65 LONG / 35 SHORT)
  3. Previous RSI in entry zone (30-55 LONG / 45-70 SHORT)
  4. RSI crosses RSI_MA + 40% TF filter + EMA filter

PARAMETERS:
  - rsi_period: 7 (source: settings.py)
  - challenge_threshold_long: 65 (source: settings.py)
  - entry_zone_long: [30, 55] (source: settings.py)

DEPENDENCIES: trend_pass.tracker, core.indicators
MIRROR: backtest/engine.py (must stay in sync with production)
```

**Step 3 — Enforce production-backtest sync**
For trading bots, production logic and backtest logic MUST be mirrors. Scan for:
- Production file: `src/worker/production_worker.py` or equivalent
- Backtest file: `backtest/engine.py` or equivalent
- Compare entry/exit function signatures and conditional branches
- Flag any divergence: "Production uses condition X but backtest doesn't"

**Step 4 — Parameter registry**
Build a parameter registry linking every configurable threshold to its source:
- Single source of truth file (e.g., `settings.py`)
- Per-token overrides (e.g., `token_config.py`, `final_config.json`)
- Scan for hardcoded magic numbers in logic files that should be in config
- Flag: "Hardcoded value 65 in detect.py:L42 — should reference settings.CHALLENGE_THRESHOLD_LONG"

**Step 5 — Strategy state machine documentation**
If the trading logic uses a multi-step state machine (e.g., 3-step RSI entry):
- Document each state and its transition conditions
- Generate a state diagram in text format
- Save to manifest as `state_machine` field on the component

```
State Machine: RSI Entry
  [IDLE] --ticket_exists--> [STEP1_CHALLENGE]
  [STEP1_CHALLENGE] --rsi_ma_crosses_threshold--> [STEP2_ZONE_CHECK]
  [STEP2_ZONE_CHECK] --prev_rsi_in_zone--> [STEP3_ENTRY_POINT]
  [STEP3_ENTRY_POINT] --rsi_crosses_rsi_ma + filters--> [SIGNAL_EMITTED]
  [any_step] --ticket_expired--> [IDLE]
```

**Step 6 — Backtest result linkage**
Link logic components to their backtest performance:
- Scan `backtest/scan_results/` or equivalent for result files
- Associate each strategy variant with its performance metrics
- Record in manifest: "RSI Entry v5 with EMA Follow: $20,445 over 6mo backtest"
- Flag if logic was modified AFTER the latest backtest: "Logic changed since last backtest — results may be invalid"

#### Example

```python
# trade-logic generates this spec from code analysis:
# COMPONENT: EMA Follow Exit
# TYPE: EXIT_LOGIC
# STATUS: ACTIVE
# BUG_HISTORY: 2026-02-22 fixed wick detection (was using close, now uses candle_low/high)
#
# EXIT CONDITION:
#   if candle_wick crosses EMA120 -> exit position
#   (NOT candle_close — this was the V4 bug)
#
# PARAMETERS:
#   ema_period: 120 (source: settings.py)
#   use_wick: True (source: settings.py, changed from False in V4)
#
# MIRROR: backtest/exit_checker.py:check_ema_follow()
# BACKTEST: $22,481 (x2.0 adaptive variant, validated 2026-02-22)
```
