---
name: "chart-components"
pack: "@rune/trading"
description: "Financial chart patterns — candlestick, line, and area charts using TradingView Lightweight Charts. Real-time update handlers, zoom, crosshair sync, indicator overlays, and responsive layout with reduced-motion support."
model: sonnet
tools: [Read, Edit, Write, Grep, Glob, Bash]
---

# chart-components

Financial chart patterns — candlestick, line, and area charts using TradingView Lightweight Charts. Real-time update handlers, zoom, crosshair sync, indicator overlays, and responsive layout with reduced-motion support.

#### Workflow

**Step 1 — Detect chart library and configure chart instance**
Use `Grep` to check for `lightweight-charts` or `@tradingview/charting_library` in `package.json`. Initialize with `createChart(container, { autoSize: true, layout: { background: { color: '#0c1419' } } })`. Create a `CandlestickSeries` with green/red up/down colors matching the project palette.

**Step 2 — Real-time update handler**
Subscribe to the normalized WebSocket feed from `realtime-data`. On each tick, call `series.update({ time, open, high, low, close, volume })`. Batch rapid updates with `requestAnimationFrame` to avoid layout thrashing. Use `Read` to verify the container element is stable (not re-mounting on every render).

**Step 3 — Responsive layout and reduced-motion**
Use `Bash` to run `window.matchMedia('(prefers-reduced-motion: reduce)')` check at init time. When true, disable chart animations (`animation: { duration: 0 }`). Add `ResizeObserver` on the container and call `chart.applyOptions({ width, height })` on size change.

#### Example

```typescript
import { createChart, CandlestickSeries } from 'lightweight-charts';

function initCandlestickChart(container: HTMLElement): CandlestickSeries {
  const reducedMotion = window.matchMedia(
    '(prefers-reduced-motion: reduce)',
  ).matches;

  const chart = createChart(container, {
    autoSize: true,
    layout: { background: { color: '#0c1419' }, textColor: '#a0aeb8' },
    grid: { vertLines: { color: '#2a3f52' }, horzLines: { color: '#2a3f52' } },
    crosshair: { mode: 1 },
    animation: { duration: reducedMotion ? 0 : 300 },
  });

  const series = chart.addSeries(CandlestickSeries, {
    upColor: '#00d084',
    downColor: '#ff6b6b',
    borderVisible: false,
    wickUpColor: '#00d084',
    wickDownColor: '#ff6b6b',
  });

  new ResizeObserver(() => chart.applyOptions({ width: container.clientWidth }))
    .observe(container);

  return series;
}
```
