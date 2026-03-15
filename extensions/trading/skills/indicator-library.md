---
name: "indicator-library"
pack: "@rune/trading"
description: "Technical indicator implementations — SMA, EMA, RSI, MACD, Bollinger Bands, VWAP. Streaming calculation patterns that update incrementally on each new tick rather than recomputing the full history."
model: sonnet
tools: [Read, Edit, Write, Grep, Glob, Bash]
---

# indicator-library

Technical indicator implementations — SMA, EMA, RSI, MACD, Bollinger Bands, VWAP. Streaming calculation patterns that update incrementally on each new tick rather than recomputing the full history.

#### Workflow

**Step 1 — Select indicators and initialize state**
Use `Read` on the product spec or existing chart config to identify required indicators. For each, allocate a rolling window buffer sized to the longest period (e.g., 200 for SMA-200). Initialize with historical OHLCV data fetched via REST before the WebSocket feed opens.

**Step 2 — Streaming incremental calculation**
On each new tick from `realtime-data`, push the close price into the rolling buffer and evict the oldest value. Recompute only the current indicator value — not the full series. For RSI, maintain running average gains/losses using Wilder smoothing. Use `Bash` to run unit tests comparing streaming output against a reference batch computation.

**Step 3 — Overlay on chart component**
Create a `LineSeries` on the chart instance from `chart-components` for each indicator. On each streaming update, call `indicatorSeries.update({ time, value })`. Use `Grep` to confirm indicator series are cleaned up (`chart.removeSeries(s)`) when the symbol or timeframe changes to prevent memory leaks.

#### Example

```typescript
class StreamingSMA {
  private readonly window: number[] = [];

  constructor(private readonly period: number) {}

  update(price: number): number | null {
    this.window.push(price);
    if (this.window.length > this.period) {
      this.window.shift();
    }
    if (this.window.length < this.period) return null;
    const sum = this.window.reduce((acc, v) => acc + v, 0);
    return sum / this.period;
  }
}

class StreamingEMA {
  private ema: number | null = null;
  private readonly k: number;

  constructor(private readonly period: number) {
    this.k = 2 / (period + 1);
  }

  update(price: number): number | null {
    this.ema = this.ema === null
      ? price
      : price * this.k + this.ema * (1 - this.k);
    return this.ema;
  }
}
```
