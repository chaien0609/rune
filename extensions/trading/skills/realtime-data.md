---
name: "realtime-data"
pack: "@rune/trading"
description: "Real-time data architecture — WebSocket lifecycle management, auto-reconnect with exponential backoff, event normalization, rate limiting, and TanStack Query cache invalidation."
model: sonnet
tools: [Read, Edit, Write, Grep, Glob, Bash]
---

# realtime-data

Real-time data architecture — WebSocket lifecycle management, auto-reconnect with exponential backoff, event normalization, rate limiting, and TanStack Query cache invalidation.

#### Workflow

**Step 1 — WebSocket setup and event normalization**
Use `Read` on existing data-fetching files to understand current polling or REST patterns. Replace with a WebSocket client class that emits typed, normalized events regardless of upstream message format. Define a `NormalizedTick` interface at the boundary.

**Step 2 — Implement exponential backoff reconnect**
In the WebSocket class, add a reconnect handler: attempt 1 after 1 s, attempt 2 after 2 s, attempt 3 after 4 s, cap at 30 s. Use `Bash` to run unit tests covering disconnect and reconnect sequences. Track `reconnectAttempts` in state; reset to 0 on successful open.

**Step 3 — Wire to TanStack Query cache invalidation**
On each normalized event received, call `queryClient.setQueryData(['ticker', symbol], tick)` for optimistic updates or `queryClient.invalidateQueries(['orderbook', symbol])` for full refresh. Use `Grep` to confirm no stale `setInterval` polling remains alongside the new WebSocket feed.

#### Example

```typescript
class TradingWebSocket {
  private ws: WebSocket | null = null;
  private reconnectAttempts = 0;
  private readonly MAX_DELAY_MS = 30_000;

  connect(url: string, onTick: (tick: NormalizedTick) => void): void {
    this.ws = new WebSocket(url);

    this.ws.onmessage = (event) => {
      const raw = JSON.parse(event.data as string);
      onTick(this.normalize(raw));
    };

    this.ws.onclose = () => {
      const delay = Math.min(
        1000 * 2 ** this.reconnectAttempts,
        this.MAX_DELAY_MS,
      );
      this.reconnectAttempts += 1;
      setTimeout(() => this.connect(url, onTick), delay);
    };

    this.ws.onopen = () => { this.reconnectAttempts = 0; };
  }

  private normalize(raw: unknown): NormalizedTick {
    // map exchange-specific shape to shared interface
    const r = raw as Record<string, unknown>;
    return { symbol: String(r['s']), price: String(r['p']), ts: Date.now() };
  }
}
```
