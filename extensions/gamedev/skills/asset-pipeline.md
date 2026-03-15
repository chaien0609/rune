---
name: "asset-pipeline"
pack: "@rune/gamedev"
description: "Game asset pipeline — glTF loading, texture compression, audio management, asset manifest, preloading."
model: sonnet
tools: [Read, Edit, Write, Grep, Glob, Bash]
---

# asset-pipeline

Game asset pipeline — glTF loading, texture compression, audio management, asset manifest, preloading.

#### Workflow

**Step 1 — Detect asset strategy**
Use Glob to find asset files: `*.gltf`, `*.glb`, `*.ktx2`, `*.basis`, `*.png` in `assets/` or `public/`. Use Grep to find loaders: `GLTFLoader`, `TextureLoader`, `KTX2Loader`, `Howler`, `Audio`. Read the loading code to understand: preloading strategy, compression, and caching.

**Step 2 — Audit asset efficiency**
Check for: uncompressed textures (PNG/JPG instead of KTX2/Basis), glTF without Draco compression, no asset manifest (scattered inline paths), missing preloader (assets load mid-gameplay causing stutters), audio files in WAV format (use OGG/MP3), and no LOD variants for 3D models.

**Step 3 — Emit asset pipeline**
Emit: asset manifest with typed entries, preloader with progress tracking, glTF loader with Draco decoder, KTX2 texture loader, and audio manager with Howler.js.

#### Example

```typescript
// Asset manifest + preloader with progress tracking
interface AssetManifest {
  models: Record<string, { url: string; draco?: boolean }>;
  textures: Record<string, { url: string; format: 'ktx2' | 'png' }>;
  audio: Record<string, { url: string; volume?: number; loop?: boolean }>;
}

const MANIFEST: AssetManifest = {
  models: {
    player: { url: '/assets/player.glb', draco: true },
    level1: { url: '/assets/level1.glb', draco: true },
  },
  textures: {
    terrain: { url: '/assets/terrain.ktx2', format: 'ktx2' },
  },
  audio: {
    bgm: { url: '/assets/bgm.ogg', volume: 0.5, loop: true },
    jump: { url: '/assets/jump.ogg', volume: 0.8 },
  },
};

class AssetLoader {
  private loaded = 0;
  private total = 0;
  private cache = new Map<string, unknown>();

  async loadAll(manifest: AssetManifest, onProgress: (pct: number) => void) {
    const entries = [
      ...Object.entries(manifest.models).map(([k, v]) => ({ key: k, ...v, type: 'model' })),
      ...Object.entries(manifest.textures).map(([k, v]) => ({ key: k, ...v, type: 'texture' })),
      ...Object.entries(manifest.audio).map(([k, v]) => ({ key: k, ...v, type: 'audio' })),
    ];
    this.total = entries.length;

    await Promise.all(entries.map(async (entry) => {
      await fetch(entry.url); // preload into browser cache
      this.loaded++;
      onProgress(this.loaded / this.total);
    }));
  }

  get<T>(key: string): T {
    const asset = this.cache.get(key);
    if (!asset) throw new Error(`Asset not loaded: ${key}`);
    return asset as T;
  }
}
```
