---
name: "particles"
pack: "@rune/gamedev"
description: "GPU particle system with WebGL instancing — object-pooled particles, emission presets, and performance-safe rendering for 10k+ particles."
model: sonnet
tools: [Read, Edit, Write, Grep, Glob, Bash]
---

# particles

GPU particle system with WebGL instancing — object-pooled particles, emission presets, and performance-safe rendering for 10k+ particles.

#### Workflow

**Step 1 — Detect particle usage**
Use Grep to find particle code: `particles`, `emitter`, `ParticleSystem`, `createBufferSource`. Understand current implementation: CPU vs GPU, pooling strategy, and draw call count.

**Step 2 — Audit particle performance**
Check for: new particle objects created on emit (GC pressure), no object pool, drawing each particle as a separate draw call, no life/alpha fade.

**Step 3 — Emit particle system**
Emit: ParticleSystem with pre-allocated pool, update/render separation, emitter presets for common effects (explosion, sparks, smoke).

#### GPU Particles with WebGL Instancing

```typescript
// GPU particle system — update on CPU, render 10k+ particles via instancing
interface Particle {
  x: number; y: number;
  vx: number; vy: number;
  life: number; maxLife: number;
  size: number;
  r: number; g: number; b: number; a: number;
}

class ParticleSystem {
  private particles: Particle[] = [];
  private pool: Particle[] = [];
  private readonly maxParticles: number;

  constructor(maxParticles = 5000) {
    this.maxParticles = maxParticles;
    // Pre-allocate pool
    for (let i = 0; i < maxParticles; i++) {
      this.pool.push({ x:0,y:0,vx:0,vy:0,life:0,maxLife:1,size:4,r:1,g:1,b:1,a:1 });
    }
  }

  emit(x: number, y: number, count: number, config: Partial<Particle> = {}) {
    for (let i = 0; i < count && this.particles.length < this.maxParticles; i++) {
      const p = this.pool.pop() ?? { x:0,y:0,vx:0,vy:0,life:0,maxLife:1,size:4,r:1,g:1,b:1,a:1 };
      const angle = Math.random() * Math.PI * 2;
      const speed = 50 + Math.random() * 150;
      Object.assign(p, {
        x, y,
        vx: Math.cos(angle) * speed,
        vy: Math.sin(angle) * speed,
        life: 1, maxLife: 0.5 + Math.random() * 1.5,
        size: 3 + Math.random() * 5,
        r: 1, g: 0.5, b: 0.1, a: 1,
        ...config,
      });
      this.particles.push(p);
    }
  }

  update(dt: number) {
    const gravity = 200;
    for (let i = this.particles.length - 1; i >= 0; i--) {
      const p = this.particles[i];
      p.x += p.vx * dt;
      p.y += p.vy * dt;
      p.vy += gravity * dt;
      p.life -= dt / p.maxLife;
      p.a = Math.max(0, p.life);
      if (p.life <= 0) {
        this.pool.push(p);
        this.particles.splice(i, 1);
      }
    }
  }

  // Render with Canvas 2D (swap with WebGL instancing for >10k particles)
  render(ctx: CanvasRenderingContext2D) {
    for (const p of this.particles) {
      ctx.globalAlpha = p.a;
      ctx.fillStyle = `rgb(${p.r * 255 | 0},${p.g * 255 | 0},${p.b * 255 | 0})`;
      ctx.beginPath();
      ctx.arc(p.x, p.y, p.size * p.life, 0, Math.PI * 2);
      ctx.fill();
    }
    ctx.globalAlpha = 1;
  }
}

// Emitter presets
const EMITTER_PRESETS = {
  explosion: (x: number, y: number, sys: ParticleSystem) =>
    sys.emit(x, y, 80, { r: 1, g: 0.4, b: 0 }),
  sparks: (x: number, y: number, sys: ParticleSystem) =>
    sys.emit(x, y, 20, { r: 1, g: 0.9, b: 0.2, size: 2, maxLife: 0.8 }),
  smoke: (x: number, y: number, sys: ParticleSystem) =>
    sys.emit(x, y, 10, { r: 0.5, g: 0.5, b: 0.5, size: 12, maxLife: 3, vx: 0, vy: -30 }),
};
```
