---
name: "threejs-patterns"
pack: "@rune/gamedev"
description: "Three.js patterns — scene setup, React Three Fiber integration, PBR materials, post-processing, performance optimization."
model: sonnet
tools: [Read, Edit, Write, Grep, Glob, Bash]
---

# threejs-patterns

Three.js patterns — scene setup, React Three Fiber integration, PBR materials, post-processing, performance optimization.

#### Workflow

**Step 1 — Detect Three.js setup**
Use Grep to find Three.js usage: `THREE.`, `useThree`, `useFrame`, `Canvas`, `@react-three/fiber`, `@react-three/drei`. Read the main scene file to understand: renderer setup, scene graph structure, camera type, and lighting model.

**Step 2 — Audit performance**
Check for: objects created inside `useFrame` (GC pressure), missing `dispose()` on unmount (memory leak), no frustum culling on large scenes, textures without power-of-two dimensions, unoptimized geometry (too many draw calls), and missing LOD for distant objects.

**Step 3 — Emit optimized scene**
Emit: properly structured R3F scene with declarative lights, memoized geometries, disposal on unmount, instanced meshes for repeated objects, and post-processing pipeline.

#### Example

```tsx
// React Three Fiber — optimized scene with instancing and post-processing
import { Canvas, useFrame } from '@react-three/fiber';
import { OrbitControls, Environment, useGLTF, Instances, Instance } from '@react-three/drei';
import { EffectComposer, Bloom, Vignette } from '@react-three/postprocessing';
import { useRef, useMemo } from 'react';

function InstancedTrees({ count = 500 }) {
  const positions = useMemo(() =>
    Array.from({ length: count }, () => [
      (Math.random() - 0.5) * 100,
      0,
      (Math.random() - 0.5) * 100,
    ] as [number, number, number]),
  [count]);

  return (
    <Instances limit={count}>
      <cylinderGeometry args={[0.2, 0.4, 3]} />
      <meshStandardMaterial color="#4a7c59" />
      {positions.map((pos, i) => <Instance key={i} position={pos} />)}
    </Instances>
  );
}

function GameScene() {
  return (
    <Canvas camera={{ position: [0, 10, 20], fov: 60 }} gl={{ antialias: true }}>
      <ambientLight intensity={0.3} />
      <directionalLight position={[10, 20, 10]} intensity={1} castShadow />
      <Environment preset="sunset" />
      <InstancedTrees count={500} />
      <OrbitControls maxPolarAngle={Math.PI / 2.2} />
      <EffectComposer>
        <Bloom intensity={0.3} luminanceThreshold={0.8} />
        <Vignette offset={0.3} darkness={0.6} />
      </EffectComposer>
    </Canvas>
  );
}
```

#### LOD (Level of Detail) Pattern

```typescript
import * as THREE from 'three';

// Swap geometry based on camera distance
function buildLODMesh(
  highGeo: THREE.BufferGeometry,
  midGeo: THREE.BufferGeometry,
  lowGeo: THREE.BufferGeometry,
  material: THREE.Material,
): THREE.LOD {
  const lod = new THREE.LOD();
  lod.addLevel(new THREE.Mesh(highGeo, material), 0);    // < 20 units
  lod.addLevel(new THREE.Mesh(midGeo, material), 20);    // 20–80 units
  lod.addLevel(new THREE.Mesh(lowGeo, material), 80);    // > 80 units
  return lod;
}

// In render loop — LOD auto-updates based on camera distance
// scene.add(buildLODMesh(highGeo, midGeo, lowGeo, mat));
// lod.update(camera); // call once per frame
```
