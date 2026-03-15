---
name: "experiment-loop"
pack: "@rune/trading"
description: "Scientific method for trading strategy development — hypothesize → implement → backtest → analyze → refine. Prevents the #1 strategy development failure: changing parameters randomly without tracking what was tested, what worked, and why."
model: sonnet
tools: [Read, Edit, Write, Grep, Glob, Bash]
---

# experiment-loop

Scientific method for trading strategy development — hypothesize → implement → backtest → analyze → refine. Prevents the #1 strategy development failure: changing parameters randomly without tracking what was tested, what worked, and why.

#### Workflow

**Step 1 — Define hypothesis**
Every strategy change starts as a falsifiable hypothesis:
```
HYPOTHESIS: [What you believe]
EVIDENCE: [Why you believe it — chart observation, backtest anomaly, market regime]
TEST: [How to verify — specific backtest config, date range, token set]
SUCCESS CRITERIA: [Measurable threshold — "win rate > 55% AND max drawdown < 15%"]
FAILURE CRITERIA: [When to reject — "win rate < 45% OR drawdown > 25%"]
```

Check `.rune/experiments/` for prior experiments on the same component. If a similar hypothesis was already tested and rejected, flag it: "This was tested in experiment #12 and rejected because [reason]. Proceed anyway?"

**Step 2 — Implement variant**
Create the strategy variant in an isolated branch or config:
- Use `Grep` to find the parameter or logic being changed
- Create a named variant (e.g., `rsi_entry_v6_longer_period`) — NEVER modify the production logic directly
- Document the exact change: "Changed RSI period from 7 to 14, challenge threshold from 65 to 60"
- If logic change (not just parameter): ensure backtest engine mirrors the change (production-backtest sync from `trade-logic`)

**Step 3 — Run backtest**
Execute backtest against the defined test conditions:
- Use `Bash` to run the backtest command with the variant config
- Capture results: total PnL, win rate, max drawdown, Sharpe ratio, number of trades
- Compare against the control (current production parameters)
- Record execution time and date range

**Step 4 — Analyze results**
Structured analysis against success/failure criteria:

```
EXPERIMENT #14: RSI Period 14 vs 7
STATUS: REJECTED ❌

RESULTS:
  | Metric        | Control (v5) | Variant (v6) | Δ       |
  |---------------|-------------|-------------|---------|
  | Total PnL     | $20,445     | $18,200     | -$2,245 |
  | Win Rate      | 58.3%       | 52.1%       | -6.2%   |
  | Max Drawdown  | 12.4%       | 14.8%       | +2.4%   |
  | Sharpe Ratio  | 1.42        | 1.18        | -0.24   |
  | Trade Count   | 156         | 89          | -67     |

CONCLUSION: Longer RSI period reduces signal frequency by 43% without
improving quality. Win rate dropped below 55% threshold. REJECTED.

INSIGHT: RSI 7 captures mean-reversion signals faster on 15m timeframe.
Longer periods may suit 4H+ timeframes (not tested — add to backlog).
```

**Step 5 — Record and route**
Save experiment to `.rune/experiments/<number>-<name>.md`:
- If **ACCEPTED**: update production parameters → run `trade-logic` to sync manifest → commit
- If **REJECTED**: record conclusion and insight → add derived hypotheses to backlog
- If **INCONCLUSIVE**: define additional test conditions or longer date range → re-run
- Link to the experiment from `trade-logic` manifest: "RSI Entry v5: validated by experiment #14"

Update experiment index `.rune/experiments/index.md`:
```
| # | Hypothesis | Component | Status | Key Metric | Date |
|---|-----------|-----------|--------|------------|------|
| 14 | RSI 14 > RSI 7 | rsi_entry | ❌ Rejected | WR 52% < 55% | 2025-03-15 |
| 13 | EMA 120 wick exit | ema_follow | ✅ Accepted | PnL +$2,036 | 2025-03-10 |
```

#### Example

```python
# Experiment runner pattern
from dataclasses import dataclass
from decimal import Decimal

@dataclass(frozen=True)
class ExperimentConfig:
    name: str
    hypothesis: str
    variant_params: dict[str, str | int | float]
    control_params: dict[str, str | int | float]
    date_range: tuple[str, str]
    tokens: list[str]
    success_criteria: dict[str, tuple[str, float]]  # metric: (operator, threshold)

@dataclass(frozen=True)
class ExperimentResult:
    config: ExperimentConfig
    control_metrics: dict[str, Decimal]
    variant_metrics: dict[str, Decimal]
    status: str  # 'accepted' | 'rejected' | 'inconclusive'
    conclusion: str
    insights: list[str]

def evaluate_experiment(result: ExperimentResult) -> str:
    """Evaluate variant against success criteria."""
    for metric, (op, threshold) in result.config.success_criteria.items():
        variant_val = result.variant_metrics.get(metric, Decimal('0'))
        if op == '>' and variant_val <= Decimal(str(threshold)):
            return 'rejected'
        if op == '<' and variant_val >= Decimal(str(threshold)):
            return 'rejected'
    return 'accepted'

# Usage:
# config = ExperimentConfig(
#     name="rsi_period_14",
#     hypothesis="RSI 14 captures better signals than RSI 7 on 15m",
#     variant_params={"rsi_period": 14, "challenge_threshold": 60},
#     control_params={"rsi_period": 7, "challenge_threshold": 65},
#     date_range=("2024-09-01", "2025-03-01"),
#     tokens=["BTCUSDT", "ETHUSDT", "SOLUSDT"],
#     success_criteria={"win_rate": (">", 0.55), "max_drawdown": ("<", 0.15)},
# )
```
