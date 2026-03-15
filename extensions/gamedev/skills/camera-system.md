---
name: "camera-system"
pack: "@rune/gamedev"
description: "Follow camera, screen shake, zoom — 2D camera with smooth lerp follow, dead zone, screen shake on impact, and zoom-to target."
model: sonnet
tools: [Read, Edit, Write, Grep, Glob, Bash]
---

# camera-system

Follow camera, screen shake, zoom — 2D camera with smooth lerp follow, dead zone, screen shake on impact, and zoom-to target.

#### Workflow

**Step 1 — Detect camera usage**
Use Grep to find camera code: `Camera`, `follow`, `viewport`, `ctx.translate`, `ctx.setTransform`. Read to understand: follow strategy, any existing shake/zoom, and how transform is applied to canvas.

**Step 2 — Audit camera configuration**
Check for: camera snapping to target each frame (no lerp = no smoothness), no dead zone (camera moves even for tiny player motion), screen shake with random offset not decaying.

**Step 3 — Emit Camera2D**
Emit: Camera2D with smooth lerp, configurable dead zone, screen shake with intensity decay, zoom-to target, and `applyToContext` helper.

#### Smooth Follow, Screen Shake, Dead Zone

```typescript
// 2D camera system — smooth follow, screen shake, zoom, dead zone
class Camera2D {
  x = 0; y = 0;
  zoom = 1;
  private targetX = 0; private targetY = 0;
  private shakeIntensity = 0; private shakeDuration = 0;
  lerpSpeed = 5;

  // Dead zone — camera only moves when target leaves this box
  private deadZone = { w: 80, h: 60 };

  follow(targetX: number, targetY: number, dt: number) {
    const dx = targetX - this.targetX;
    const dy = targetY - this.targetY;

    // Only move camera when target exits dead zone
    if (Math.abs(dx) > this.deadZone.w / 2) this.targetX += dx - Math.sign(dx) * this.deadZone.w / 2;
    if (Math.abs(dy) > this.deadZone.h / 2) this.targetY += dy - Math.sign(dy) * this.deadZone.h / 2;

    // Smooth lerp
    this.x += (this.targetX - this.x) * this.lerpSpeed * dt;
    this.y += (this.targetY - this.y) * this.lerpSpeed * dt;
  }

  shake(intensity: number, duration: number) {
    this.shakeIntensity = intensity;
    this.shakeDuration = duration;
  }

  zoomTo(target: number, dt: number, speed = 3) {
    this.zoom += (target - this.zoom) * speed * dt;
  }

  update(dt: number): { x: number; y: number; zoom: number } {
    let sx = 0; let sy = 0;
    if (this.shakeDuration > 0) {
      this.shakeDuration -= dt;
      const t = Math.max(0, this.shakeDuration);
      sx = (Math.random() * 2 - 1) * this.shakeIntensity * t;
      sy = (Math.random() * 2 - 1) * this.shakeIntensity * t;
    }
    return { x: this.x + sx, y: this.y + sy, zoom: this.zoom };
  }

  // Apply to Canvas 2D context
  applyToContext(ctx: CanvasRenderingContext2D, screenW: number, screenH: number) {
    const { x, y, zoom } = this.update(1 / 60);
    ctx.setTransform(zoom, 0, 0, zoom, screenW / 2 - x * zoom, screenH / 2 - y * zoom);
  }
}

// Usage
const camera = new Camera2D();
camera.lerpSpeed = 8;

// In game loop render:
// camera.follow(player.x, player.y, dt);
// camera.applyToContext(ctx, canvas.width, canvas.height);
// ... draw scene ...
// ctx.setTransform(1, 0, 0, 1, 0, 0); // reset for HUD
```
