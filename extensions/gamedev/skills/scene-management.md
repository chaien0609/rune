---
name: "scene-management"
pack: "@rune/gamedev"
description: "Scene transitions, preloading, serialization — stack-based SceneManager with fade transitions, asset preloading before enter, and level JSON serialization."
model: sonnet
tools: [Read, Edit, Write, Grep, Glob, Bash]
---

# scene-management

Scene transitions, preloading, serialization — stack-based SceneManager with fade transitions, asset preloading before enter, and level JSON serialization.

#### Workflow

**Step 1 — Detect scene structure**
Use Grep to find scene code: `Scene`, `GameState`, `StateMachine`, `push`, `pop`, `replace`. Read to understand: how scenes transition, whether assets are preloaded, and how scene data is persisted.

**Step 2 — Audit scene management**
Check for: abrupt scene switches with no transition (jarring UX), assets loaded mid-scene causing stutters, no scene stack (can't return to previous state), missing cleanup on exit.

**Step 3 — Emit SceneManager**
Emit: stack-based SceneManager with fade in/out, asset preloading tied to scene.assets[], push/pop/replace operations, and level serialization/deserialization.

#### Scene Stack with Transitions and Preloading

```typescript
// Scene manager — stack-based, asset preloading, fade transitions
interface Scene {
  name: string;
  assets: string[]; // asset keys to preload before entering
  onEnter(data?: unknown): void;
  onExit(): void;
  update(dt: number): void;
  render(ctx: CanvasRenderingContext2D): void;
}

class SceneManager {
  private stack: Scene[] = [];
  private loader: AssetLoader;
  private transitioning = false;
  private fadeAlpha = 0;
  private fadeDir: 1 | -1 = 1;

  constructor(loader: AssetLoader) { this.loader = loader; }

  get current(): Scene | undefined { return this.stack[this.stack.length - 1]; }

  async push(scene: Scene, data?: unknown) {
    if (this.transitioning) return;
    this.transitioning = true;

    await this.fadeOut();
    await this.preloadScene(scene);
    this.stack.push(scene);
    scene.onEnter(data);
    await this.fadeIn();

    this.transitioning = false;
  }

  async pop() {
    if (this.transitioning || this.stack.length <= 1) return;
    this.transitioning = true;

    await this.fadeOut();
    this.current?.onExit();
    this.stack.pop();
    await this.fadeIn();

    this.transitioning = false;
  }

  async replace(scene: Scene, data?: unknown) {
    if (this.transitioning) return;
    this.transitioning = true;

    await this.fadeOut();
    this.current?.onExit();
    this.stack.pop();
    await this.preloadScene(scene);
    this.stack.push(scene);
    scene.onEnter(data);
    await this.fadeIn();

    this.transitioning = false;
  }

  private async preloadScene(scene: Scene) {
    // Preload only assets not already cached
    await Promise.all(scene.assets.map(key => this.loader.load(key, `/assets/${key}`)));
  }

  renderTransition(ctx: CanvasRenderingContext2D, w: number, h: number) {
    if (this.fadeAlpha > 0) {
      ctx.save();
      ctx.globalAlpha = this.fadeAlpha;
      ctx.fillStyle = '#000';
      ctx.fillRect(0, 0, w, h);
      ctx.restore();
    }
  }

  private fadeOut(): Promise<void> {
    return new Promise(resolve => {
      this.fadeDir = 1;
      const interval = setInterval(() => {
        this.fadeAlpha = Math.min(1, this.fadeAlpha + 0.05);
        if (this.fadeAlpha >= 1) { clearInterval(interval); resolve(); }
      }, 16);
    });
  }

  private fadeIn(): Promise<void> {
    return new Promise(resolve => {
      this.fadeDir = -1;
      const interval = setInterval(() => {
        this.fadeAlpha = Math.max(0, this.fadeAlpha - 0.05);
        if (this.fadeAlpha <= 0) { clearInterval(interval); resolve(); }
      }, 16);
    });
  }
}

// Level serialization — save/load level state as JSON
interface LevelData {
  name: string;
  entities: Array<{ type: string; x: number; y: number; props: Record<string, unknown> }>;
  tilemap: number[][];
}

function serializeLevel(world: World): LevelData {
  const entities: LevelData['entities'] = [];
  for (const id of world.query(Position)) {
    const pos = world.getComponent(id, Position)!;
    entities.push({ type: 'generic', x: pos.x, y: pos.y, props: {} });
  }
  return { name: 'level1', entities, tilemap: [] };
}

function deserializeLevel(data: LevelData, world: World) {
  for (const e of data.entities) {
    const id = world.createEntity();
    world.addComponent(id, new Position(e.x, e.y));
  }
}
```
