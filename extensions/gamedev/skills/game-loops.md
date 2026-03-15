---
name: "game-loops"
pack: "@rune/gamedev"
description: "Game loop architecture — fixed timestep, interpolation, input handling, state machines, ECS."
model: sonnet
tools: [Read, Edit, Write, Grep, Glob, Bash]
---

# game-loops

Game loop architecture — fixed timestep, interpolation, input handling, state machines, ECS.

#### Workflow

**Step 1 — Detect game loop pattern**
Use Grep to find loop code: `requestAnimationFrame`, `setInterval.*16`, `update`, `fixedUpdate`, `deltaTime`, `gameLoop`. Read the main loop to understand: timestep strategy, update/render separation, and input handling.

**Step 2 — Audit loop correctness**
Check for: variable timestep physics (non-deterministic), no accumulator for fixed update (physics tied to framerate), input polled inside render (inconsistent), missing interpolation between fixed steps (visual stuttering), and no frame budget monitoring.

**Step 3 — Emit fixed timestep loop**
Emit: fixed timestep (60Hz) with accumulator, interpolation for smooth rendering, decoupled input handler, and frame budget monitoring.

#### Example

```typescript
// Fixed timestep game loop with interpolation
const TICK_RATE = 60;
const TICK_DURATION = 1000 / TICK_RATE;

class GameLoop {
  private accumulator = 0;
  private previousTime = 0;
  private running = false;

  constructor(
    private update: (dt: number) => void,     // fixed timestep logic
    private render: (alpha: number) => void,   // interpolated rendering
  ) {}

  start() {
    this.running = true;
    this.previousTime = performance.now();
    requestAnimationFrame(this.tick);
  }

  private tick = (currentTime: number) => {
    if (!this.running) return;
    const elapsed = Math.min(currentTime - this.previousTime, 250); // cap spiral of death
    this.previousTime = currentTime;
    this.accumulator += elapsed;

    while (this.accumulator >= TICK_DURATION) {
      this.update(TICK_DURATION / 1000); // dt in seconds
      this.accumulator -= TICK_DURATION;
    }

    const alpha = this.accumulator / TICK_DURATION; // interpolation factor [0, 1)
    this.render(alpha);
    requestAnimationFrame(this.tick);
  };

  stop() { this.running = false; }
}

// Usage
const loop = new GameLoop(
  (dt) => { world.step(dt); entities.forEach(e => e.update(dt)); },
  (alpha) => { renderer.render(scene, camera, alpha); },
);
loop.start();
```
