---
name: "ecs"
pack: "@rune/gamedev"
description: "Entity Component System architecture — archetype-based ECS with dense array storage for cache efficiency, pure system functions, and query-based entity iteration."
model: sonnet
tools: [Read, Edit, Write, Grep, Glob, Bash]
---

# ecs

Entity Component System architecture — archetype-based ECS with dense array storage for cache efficiency, pure system functions, and query-based entity iteration.

#### Workflow

**Step 1 — Detect ECS usage**
Use Grep to find ECS patterns: `Entity`, `Component`, `System`, `World`, `bitECS`, `createWorld`. Read entity management code to understand: component storage, system execution order, and query patterns.

**Step 2 — Audit ECS architecture**
Check for: logic mixed into components (components should be pure data), O(n²) entity iteration without spatial partitioning, components stored as objects (prefer flat arrays for cache efficiency), and missing entity cleanup on destroy.

**Step 3 — Emit ECS scaffold**
Emit: World class with entity registry + component storage, query-based entity iteration, pure system functions, and proper cleanup.

#### Lightweight ECS — Archetype-Based

```typescript
// Minimal ECS — dense array storage per archetype for cache efficiency
type EntityId = number;
type ComponentType<T> = { new(...args: unknown[]): T; readonly typeName: string; };

// Components are plain data, no logic
class Position { static typeName = 'Position'; constructor(public x = 0, public y = 0) {} }
class Velocity { static typeName = 'Velocity'; constructor(public vx = 0, public vy = 0) {} }
class Sprite   { static typeName = 'Sprite';   constructor(public textureId = '') {} }
class Health   { static typeName = 'Health';   constructor(public hp = 100, public max = 100) {} }

// World — entity registry + component storage
class World {
  private nextId = 1;
  private components = new Map<string, Map<EntityId, unknown>>();

  createEntity(): EntityId { return this.nextId++; }

  addComponent<T extends object>(entity: EntityId, component: T & { constructor: { typeName: string } }) {
    const name = component.constructor.typeName;
    if (!this.components.has(name)) this.components.set(name, new Map());
    this.components.get(name)!.set(entity, component);
  }

  getComponent<T>(entity: EntityId, type: ComponentType<T>): T | undefined {
    return this.components.get(type.typeName)?.get(entity) as T | undefined;
  }

  removeComponent<T>(entity: EntityId, type: ComponentType<T>) {
    this.components.get(type.typeName)?.delete(entity);
  }

  // Query — iterate entities that have ALL specified components
  query<T extends object[]>(...types: { [K in keyof T]: ComponentType<T[K]> }): EntityId[] {
    if (types.length === 0) return [];
    const [first, ...rest] = types;
    const candidates = Array.from(this.components.get(first.typeName)?.keys() ?? []);
    return candidates.filter(id => rest.every(t => this.components.get(t.typeName)?.has(id)));
  }

  destroyEntity(entity: EntityId) {
    this.components.forEach(store => store.delete(entity));
  }
}

// Systems — pure functions over component queries
const movementSystem = (world: World, dt: number) => {
  for (const id of world.query(Position, Velocity)) {
    const pos = world.getComponent(id, Position)!;
    const vel = world.getComponent(id, Velocity)!;
    pos.x += vel.vx * dt;
    pos.y += vel.vy * dt;
  }
};

const healthSystem = (world: World) => {
  for (const id of world.query(Health)) {
    const hp = world.getComponent(id, Health)!;
    if (hp.hp <= 0) world.destroyEntity(id);
  }
};

// Usage
const world = new World();
const player = world.createEntity();
world.addComponent(player, new Position(100, 100));
world.addComponent(player, new Velocity(0, 0));
world.addComponent(player, new Health(100, 100));

// In fixed update:
// movementSystem(world, dt);
// healthSystem(world);
```
