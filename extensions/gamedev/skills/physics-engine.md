---
name: "physics-engine"
pack: "@rune/gamedev"
description: "Physics integration — Rapier.js, rigid bodies, constraints, raycasting, collision callbacks, deterministic simulation."
model: sonnet
tools: [Read, Edit, Write, Grep, Glob, Bash]
---

# physics-engine

Physics integration — Rapier.js, rigid bodies, constraints, raycasting, collision callbacks, deterministic simulation.

#### Workflow

**Step 1 — Detect physics setup**
Use Grep to find physics libraries: `rapier`, `cannon`, `ammo`, `@dimforge/rapier3d`, `RigidBody`, `Collider`. Read physics initialization and body creation to understand: engine choice, world configuration, and collision handling.

**Step 2 — Audit physics configuration**
Check for: physics step tied to render frame (non-deterministic), missing collision groups (everything collides with everything), no sleep threshold (wasted CPU on static objects), raycasts without max distance (expensive), and missing body cleanup on entity destroy.

**Step 3 — Emit optimized physics**
Emit: Rapier.js (WASM, deterministic) setup with proper collision groups, sleep thresholds, event-driven collision callbacks, and raycasting utility.

#### Example

```typescript
// Rapier.js (WASM) — setup with collision groups and raycasting
import RAPIER from '@dimforge/rapier3d-compat';

await RAPIER.init();
const world = new RAPIER.World({ x: 0, y: -9.81, z: 0 });

// Collision groups: player=0x0001, enemy=0x0002, ground=0x0004, projectile=0x0008
const GROUPS = { PLAYER: 0x0001, ENEMY: 0x0002, GROUND: 0x0004, PROJECTILE: 0x0008 };

// Ground — static, collides with everything
const groundBody = world.createRigidBody(RAPIER.RigidBodyDesc.fixed().setTranslation(0, 0, 0));
world.createCollider(
  RAPIER.ColliderDesc.cuboid(50, 0.1, 50)
    .setCollisionGroups((GROUPS.GROUND << 16) | 0xFFFF),
  groundBody,
);

// Player — dynamic, collides with ground + enemy (not own projectiles)
const playerBody = world.createRigidBody(RAPIER.RigidBodyDesc.dynamic().setTranslation(0, 5, 0));
world.createCollider(
  RAPIER.ColliderDesc.capsule(0.5, 0.3)
    .setCollisionGroups((GROUPS.PLAYER << 16) | (GROUPS.GROUND | GROUPS.ENEMY))
    .setActiveEvents(RAPIER.ActiveEvents.COLLISION_EVENTS),
  playerBody,
);

// Raycast utility
function raycast(origin: RAPIER.Vector3, direction: RAPIER.Vector3, maxDist = 100) {
  const ray = new RAPIER.Ray(origin, direction);
  const hit = world.castRay(ray, maxDist, true);
  if (hit) {
    const point = ray.pointAt(hit.timeOfImpact);
    return { point, normal: hit.normal, collider: hit.collider };
  }
  return null;
}
```

#### Collision Event Handling

```typescript
// Event-driven collision callbacks — no polling
const eventQueue = new RAPIER.EventQueue(true);

// In fixed update step:
world.step(eventQueue);

eventQueue.drainCollisionEvents((handle1, handle2, started) => {
  const body1 = world.getRigidBody(world.getCollider(handle1).parent()!);
  const body2 = world.getRigidBody(world.getCollider(handle2).parent()!);

  const entity1 = entityMap.get(body1.handle);
  const entity2 = entityMap.get(body2.handle);

  if (started) {
    entity1?.onCollisionEnter(entity2);
    entity2?.onCollisionEnter(entity1);
  } else {
    entity1?.onCollisionExit(entity2);
    entity2?.onCollisionExit(entity1);
  }
});
```
