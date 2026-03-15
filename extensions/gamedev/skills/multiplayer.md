---
name: "multiplayer"
pack: "@rune/gamedev"
description: "WebSocket game server and client prediction — authoritative server model, client-side prediction, reconciliation, entity interpolation, and lag compensation."
model: sonnet
tools: [Read, Edit, Write, Grep, Glob, Bash]
---

# multiplayer

WebSocket game server and client prediction — authoritative server model, client-side prediction, reconciliation, entity interpolation, and lag compensation.

Authoritative server model: server owns game state, clients send inputs only. Never trust client position.

#### Workflow

**Step 1 — Assess multiplayer architecture**
Use Grep to find existing WebSocket code: `WebSocket`, `socket.io`, `ws`, `onmessage`, `postMessage`. Determine: is there a server or is this client-only? What is the tick rate?

**Step 2 — Implement authoritative server**
Emit: Node.js WebSocket server with fixed-tick update (20Hz sufficient), input queue per player, server-side physics/movement, state broadcast.

**Step 3 — Implement client prediction**
Emit: local input application before server confirmation, pending input buffer, reconciliation on server update, smooth interpolation for remote entities.

**Step 4 — Add lag compensation**
Emit: snapshot buffer for entity interpolation (render ~100ms behind server), client prediction with rollback on desync.

#### WebSocket Game Server Pattern

```typescript
// Server (Node.js + ws) — authoritative game server
import { WebSocketServer, WebSocket } from 'ws';

interface PlayerInput { seq: number; keys: { up: boolean; down: boolean; left: boolean; right: boolean }; }
interface PlayerState { id: string; x: number; y: number; vx: number; vy: number; }

const wss = new WebSocketServer({ port: 3001 });
const players = new Map<string, PlayerState>();
const inputQueues = new Map<string, PlayerInput[]>();

wss.on('connection', (ws: WebSocket, req) => {
  const id = crypto.randomUUID();
  players.set(id, { id, x: 0, y: 0, vx: 0, vy: 0 });
  inputQueues.set(id, []);

  ws.send(JSON.stringify({ type: 'init', id, state: Object.fromEntries(players) }));
  broadcast({ type: 'player_joined', id });

  ws.on('message', (raw) => {
    const msg = JSON.parse(raw.toString());
    if (msg.type === 'input') {
      inputQueues.get(id)?.push(msg.input);
    }
  });

  ws.on('close', () => {
    players.delete(id);
    inputQueues.delete(id);
    broadcast({ type: 'player_left', id });
  });
});

// Fixed-tick server update (20Hz is sufficient for authoritative server)
const TICK_MS = 50;
setInterval(() => {
  inputQueues.forEach((queue, id) => {
    const player = players.get(id)!;
    const input = queue.shift(); // process one input per tick
    if (input) applyInput(player, input);
    integratePhysics(player);
  });

  broadcast({ type: 'state_update', tick: Date.now(), players: Object.fromEntries(players) });
}, TICK_MS);

function broadcast(msg: object) {
  const data = JSON.stringify(msg);
  wss.clients.forEach(c => c.readyState === WebSocket.OPEN && c.send(data));
}

function applyInput(p: PlayerState, input: PlayerInput) {
  const speed = 200;
  p.vx = (input.keys.right ? 1 : 0) - (input.keys.left ? 1 : 0);
  p.vy = (input.keys.down ? 1 : 0) - (input.keys.up ? 1 : 0);
  const len = Math.hypot(p.vx, p.vy);
  if (len > 0) { p.vx = (p.vx / len) * speed; p.vy = (p.vy / len) * speed; }
}

function integratePhysics(p: PlayerState) {
  p.x += p.vx * (TICK_MS / 1000);
  p.y += p.vy * (TICK_MS / 1000);
}
```

#### Client Prediction & Reconciliation

```typescript
// Client — predict locally, reconcile on server update
class NetworkedPlayer {
  private pendingInputs: { seq: number; input: PlayerInput; }[] = [];
  private seq = 0;
  localState: PlayerState;
  serverState: PlayerState;

  constructor(initial: PlayerState) {
    this.localState = { ...initial };
    this.serverState = { ...initial };
  }

  sendInput(keys: PlayerInput['keys'], ws: WebSocket) {
    const input: PlayerInput = { seq: ++this.seq, keys };
    this.pendingInputs.push({ seq: this.seq, input });
    ws.send(JSON.stringify({ type: 'input', input }));

    // Apply immediately (client prediction)
    applyInputToState(this.localState, keys);
  }

  reconcile(serverUpdate: PlayerState & { lastProcessedSeq: number }) {
    this.serverState = serverUpdate;

    // Remove acknowledged inputs
    this.pendingInputs = this.pendingInputs.filter(p => p.seq > serverUpdate.lastProcessedSeq);

    // Reapply unacknowledged inputs on top of server state
    this.localState = { ...serverUpdate };
    for (const { input } of this.pendingInputs) {
      applyInputToState(this.localState, input.keys);
    }
  }
}

function applyInputToState(state: PlayerState, keys: PlayerInput['keys']) {
  const dt = 1 / 60;
  const speed = 200;
  state.x += ((keys.right ? 1 : 0) - (keys.left ? 1 : 0)) * speed * dt;
  state.y += ((keys.down ? 1 : 0) - (keys.up ? 1 : 0)) * speed * dt;
}
```

#### Lag Compensation & Entity Interpolation

```typescript
// Interpolate remote entities between server snapshots (smooth movement, ~100ms behind)
interface Snapshot { tick: number; timestamp: number; entities: Map<string, PlayerState>; }

class EntityInterpolator {
  private buffer: Snapshot[] = [];
  private readonly delay = 100; // ms behind server

  addSnapshot(snapshot: Snapshot) {
    this.buffer.push(snapshot);
    // Keep only last 1 second of snapshots
    const cutoff = Date.now() - 1000;
    this.buffer = this.buffer.filter(s => s.timestamp > cutoff);
  }

  getInterpolatedState(entityId: string): PlayerState | null {
    const renderTime = Date.now() - this.delay;

    // Find the two snapshots bracketing renderTime
    const newer = this.buffer.find(s => s.timestamp >= renderTime);
    const older = this.buffer.slice().reverse().find(s => s.timestamp < renderTime);

    if (!older || !newer) return newer?.entities.get(entityId) ?? null;

    const t = (renderTime - older.timestamp) / (newer.timestamp - older.timestamp);
    const a = older.entities.get(entityId);
    const b = newer.entities.get(entityId);
    if (!a || !b) return null;

    return {
      ...a,
      x: a.x + (b.x - a.x) * t,
      y: a.y + (b.y - a.y) * t,
    };
  }
}
```
