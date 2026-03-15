---
name: "input-system"
pack: "@rune/gamedev"
description: "Keyboard/mouse/gamepad/touch input handling — unified InputManager with action mapping, input buffering, coyote time, and touch virtual joystick."
model: sonnet
tools: [Read, Edit, Write, Grep, Glob, Bash]
---

# input-system

Keyboard/mouse/gamepad/touch input handling — unified InputManager with action mapping, input buffering, coyote time, and touch virtual joystick.

#### Workflow

**Step 1 — Detect input handling**
Use Grep to find input code: `addEventListener.*keydown`, `addEventListener.*gamepad`, `onMouseMove`, `ontouchstart`. Read input handlers to understand: action mapping strategy, polling vs event-driven, and platform support.

**Step 2 — Audit input correctness**
Check for: input polled inside render loop (should be in fixed update), no gamepad support, missing touch controls for mobile, hardcoded key bindings, and no input buffering (missed jump inputs).

**Step 3 — Emit InputManager**
Emit: unified InputManager with keyboard/mouse/gamepad/touch, action-based API, justPressed/wasReleased one-frame flags, and runtime rebinding. Always call `flush()` at end of each fixed tick.

#### Unified Input Handler — Keyboard, Mouse, Gamepad, Touch

```typescript
// InputManager — supports keyboard, mouse, gamepad, touch with action mapping
type ActionMap = Record<string, string[]>; // action → [key1, key2, ...]

const DEFAULT_ACTIONS: ActionMap = {
  moveUp:    ['KeyW', 'ArrowUp'],
  moveDown:  ['KeyS', 'ArrowDown'],
  moveLeft:  ['KeyA', 'ArrowLeft'],
  moveRight: ['KeyD', 'ArrowRight'],
  jump:      ['Space'],
  attack:    ['Mouse0'],
  pause:     ['Escape'],
};

class InputManager {
  private held = new Set<string>();
  private justPressed = new Set<string>();
  private justReleased = new Set<string>();
  private axes = { x: 0, y: 0 };
  private gamepad: Gamepad | null = null;
  private bindings: ActionMap;

  constructor(bindings = DEFAULT_ACTIONS) {
    this.bindings = { ...bindings };
    window.addEventListener('keydown', e => { this.held.add(e.code); this.justPressed.add(e.code); });
    window.addEventListener('keyup', e => { this.held.delete(e.code); this.justReleased.add(e.code); });
    window.addEventListener('mousedown', e => this.held.add(`Mouse${e.button}`));
    window.addEventListener('mouseup', e => this.held.delete(`Mouse${e.button}`));
    window.addEventListener('gamepaddisconnected', () => { this.gamepad = null; });
  }

  // Call at the START of each fixed update tick, not inside render
  pollGamepad() {
    const pads = navigator.getGamepads();
    this.gamepad = pads[0] ?? null;
    if (this.gamepad) {
      this.axes.x = this.gamepad.axes[0];
      this.axes.y = this.gamepad.axes[1];
    }
  }

  // Call at END of each fixed update tick to clear one-frame flags
  flush() {
    this.justPressed.clear();
    this.justReleased.clear();
  }

  isDown(action: string): boolean {
    return (this.bindings[action] ?? []).some(key => this.held.has(key));
  }

  wasPressed(action: string): boolean {
    return (this.bindings[action] ?? []).some(key => this.justPressed.has(key));
  }

  wasReleased(action: string): boolean {
    return (this.bindings[action] ?? []).some(key => this.justReleased.has(key));
  }

  getAxes() { return { ...this.axes }; }

  // Rebind at runtime (save to localStorage)
  rebind(action: string, keys: string[]) {
    this.bindings[action] = keys;
    localStorage.setItem('inputBindings', JSON.stringify(this.bindings));
  }

  loadSavedBindings() {
    const saved = localStorage.getItem('inputBindings');
    if (saved) this.bindings = { ...this.bindings, ...JSON.parse(saved) };
  }
}
```

#### Input Buffering (Coyote Time + Jump Buffering)

```typescript
// Input buffer — remember button press for N frames to forgive missed timing
class InputBuffer {
  private buffer: Map<string, number> = new Map(); // action → frames remaining

  bufferAction(action: string, frames = 6) { this.buffer.set(action, frames); }

  consume(action: string): boolean {
    if ((this.buffer.get(action) ?? 0) > 0) {
      this.buffer.set(action, 0);
      return true;
    }
    return false;
  }

  tick() {
    this.buffer.forEach((frames, action) => {
      if (frames > 0) this.buffer.set(action, frames - 1);
    });
  }
}

// Coyote time — allow jump for N frames after walking off a ledge
class CoyoteTime {
  private grounded = false;
  private coyoteFrames = 0;
  private readonly maxFrames = 6;

  update(isGrounded: boolean) {
    if (isGrounded) {
      this.grounded = true;
      this.coyoteFrames = this.maxFrames;
    } else {
      this.grounded = false;
      this.coyoteFrames = Math.max(0, this.coyoteFrames - 1);
    }
  }

  canJump(): boolean { return this.coyoteFrames > 0; }
}
```

#### Touch Virtual Joystick (Mobile)

```typescript
// Canvas-rendered virtual joystick for mobile
class VirtualJoystick {
  private origin: { x: number; y: number } | null = null;
  private current: { x: number; y: number } | null = null;
  private touchId: number | null = null;
  readonly radius = 60;

  constructor(private canvas: HTMLCanvasElement) {
    canvas.addEventListener('touchstart', this.onStart, { passive: false });
    canvas.addEventListener('touchmove', this.onMove, { passive: false });
    canvas.addEventListener('touchend', this.onEnd);
  }

  private onStart = (e: TouchEvent) => {
    e.preventDefault();
    if (this.touchId !== null) return;
    const t = e.changedTouches[0];
    this.touchId = t.identifier;
    this.origin = { x: t.clientX, y: t.clientY };
    this.current = { ...this.origin };
  };

  private onMove = (e: TouchEvent) => {
    e.preventDefault();
    const t = Array.from(e.changedTouches).find(t => t.identifier === this.touchId);
    if (!t || !this.origin) return;
    const dx = t.clientX - this.origin.x;
    const dy = t.clientY - this.origin.y;
    const len = Math.hypot(dx, dy);
    const clamped = Math.min(len, this.radius);
    const angle = Math.atan2(dy, dx);
    this.current = {
      x: this.origin.x + Math.cos(angle) * clamped,
      y: this.origin.y + Math.sin(angle) * clamped,
    };
  };

  private onEnd = (e: TouchEvent) => {
    if (Array.from(e.changedTouches).some(t => t.identifier === this.touchId)) {
      this.origin = this.current = null;
      this.touchId = null;
    }
  };

  getAxes(): { x: number; y: number } {
    if (!this.origin || !this.current) return { x: 0, y: 0 };
    return {
      x: (this.current.x - this.origin.x) / this.radius,
      y: (this.current.y - this.origin.y) / this.radius,
    };
  }
}
```
