---
name: "quant-analysis"
pack: "@rune/trading"
description: "Quantitative analysis patterns — portfolio metrics, risk calculations, statistical edge detection, Monte Carlo simulation, and position sizing models."
model: sonnet
tools: [Read, Edit, Write, Grep, Glob, Bash]
---

# quant-analysis

Quantitative analysis patterns — portfolio metrics, risk calculations, statistical edge detection, Monte Carlo simulation, and position sizing models.

#### Workflow

**Step 1 — Define analysis scope**
Determine what the user needs: portfolio-level metrics (Sharpe, Sortino, max drawdown, VaR), strategy-level analysis (win rate, profit factor, expectancy, risk-of-ruin), or position sizing (Kelly criterion, fixed fractional, volatility-adjusted). Load trade history from data source (CSV, database query, API response).

**Step 2 — Calculate core metrics**
For portfolio analysis:
- **Sharpe Ratio**: (mean return - risk-free rate) / std(returns). Annualize with √252.
- **Sortino Ratio**: (mean return - risk-free rate) / downside_std. Only penalizes downside volatility.
- **Max Drawdown**: Largest peak-to-trough decline. Include recovery time.
- **Value at Risk (VaR)**: 95th/99th percentile loss using historical simulation or parametric method.
- **Calmar Ratio**: Annualized return / max drawdown. > 1.0 = good risk-adjusted return.

For strategy analysis:
- **Expectancy**: (win_rate × avg_win) - (loss_rate × avg_loss). Must be positive.
- **Profit Factor**: gross_profit / gross_loss. > 1.5 = viable, > 2.0 = strong.
- **Risk of Ruin**: probability of losing X% of capital given win rate and risk per trade.

**Step 3 — Monte Carlo simulation**
Run 10,000 random resamples of the trade sequence to estimate:
- Probability of reaching profit target within N trades
- Confidence interval for max drawdown (95th percentile)
- Optimal position size that maximizes geometric growth (Kelly fraction)

Emit results as structured data + visualization-ready format for `chart-components`.

**Step 4 — Position sizing recommendation**
Based on Monte Carlo results, recommend:
- **Conservative**: Half-Kelly (50% of optimal Kelly fraction)
- **Moderate**: Full Kelly
- **Aggressive**: 1.5x Kelly (with warning about increased ruin probability)

Save analysis to `.rune/trading/quant-analysis-<date>.md`.

#### Example

```typescript
import Decimal from 'decimal.js';

interface TradeResult {
  pnl: Decimal;
  entryPrice: Decimal;
  exitPrice: Decimal;
  size: Decimal;
  duration: number; // minutes
}

interface QuantMetrics {
  totalTrades: number;
  winRate: Decimal;
  profitFactor: Decimal;
  expectancy: Decimal;
  sharpeRatio: Decimal;
  sortinoRatio: Decimal;
  maxDrawdown: Decimal;
  maxDrawdownDuration: number;
  calmarRatio: Decimal;
  valueAtRisk95: Decimal;
  kellyFraction: Decimal;
  riskOfRuin: Decimal;
}

function calculateExpectancy(trades: TradeResult[]): Decimal {
  const wins = trades.filter(t => t.pnl.gt(0));
  const losses = trades.filter(t => t.pnl.lte(0));
  const winRate = new Decimal(wins.length).div(trades.length);
  const avgWin = wins.length > 0
    ? wins.reduce((sum, t) => sum.plus(t.pnl), new Decimal(0)).div(wins.length)
    : new Decimal(0);
  const avgLoss = losses.length > 0
    ? losses.reduce((sum, t) => sum.plus(t.pnl.abs()), new Decimal(0)).div(losses.length)
    : new Decimal(0);
  return winRate.mul(avgWin).minus(new Decimal(1).minus(winRate).mul(avgLoss));
}

function kellyFraction(winRate: Decimal, avgWinLossRatio: Decimal): Decimal {
  // Kelly: f* = (p * b - q) / b where p=winRate, q=1-p, b=avgWin/avgLoss
  const q = new Decimal(1).minus(winRate);
  return winRate.mul(avgWinLossRatio).minus(q).div(avgWinLossRatio);
}

// Monte Carlo: resample trades 10,000 times
function monteCarloDrawdown(trades: TradeResult[], simulations = 10000): Decimal {
  const drawdowns: Decimal[] = [];
  for (let i = 0; i < simulations; i++) {
    const shuffled = [...trades].sort(() => Math.random() - 0.5);
    let peak = new Decimal(0), maxDd = new Decimal(0), equity = new Decimal(0);
    for (const t of shuffled) {
      equity = equity.plus(t.pnl);
      if (equity.gt(peak)) peak = equity;
      const dd = peak.minus(equity).div(peak.gt(0) ? peak : new Decimal(1));
      if (dd.gt(maxDd)) maxDd = dd;
    }
    drawdowns.push(maxDd);
  }
  drawdowns.sort((a, b) => a.cmp(b));
  return drawdowns[Math.floor(simulations * 0.95)]; // 95th percentile
}
```
