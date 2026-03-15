---
name: "audio-system"
pack: "@rune/gamedev"
description: "Web Audio API, spatial audio, SFX management — AudioManager with spatial audio, music crossfade, and SFX pooling."
model: sonnet
tools: [Read, Edit, Write, Grep, Glob, Bash]
---

# audio-system

Web Audio API, spatial audio, SFX management — AudioManager with spatial audio, music crossfade, and SFX pooling.

#### Workflow

**Step 1 — Detect audio setup**
Use Grep to find audio code: `AudioContext`, `Howler`, `Audio`, `createBufferSource`, `createGain`. Read the audio initialization to understand: context management, volume controls, and loading strategy.

**Step 2 — Audit audio configuration**
Check for: AudioContext not resumed on user gesture (browser autoplay policy), SFX creating new nodes on every play (memory pressure), no volume normalization (clipping), missing cleanup on scene exit.

**Step 3 — Emit AudioManager**
Emit: AudioManager class with master/music/sfx gain chain, music crossfade, spatial (3D) audio via PannerNode, and SFX pooling.

#### Web Audio API — Full Audio Manager

```typescript
// AudioManager — spatial audio, music crossfade, SFX pooling
class AudioManager {
  private ctx: AudioContext;
  private masterGain: GainNode;
  private musicGain: GainNode;
  private sfxGain: GainNode;
  private buffers = new Map<string, AudioBuffer>();
  private sfxPool = new Map<string, AudioBufferSourceNode[]>();
  private currentMusic: AudioBufferSourceNode | null = null;

  constructor() {
    this.ctx = new AudioContext();
    this.masterGain = this.ctx.createGain();
    this.musicGain = this.ctx.createGain();
    this.sfxGain = this.ctx.createGain();

    this.musicGain.connect(this.masterGain);
    this.sfxGain.connect(this.masterGain);
    this.masterGain.connect(this.ctx.destination);

    this.musicGain.gain.value = 0.6;
    this.sfxGain.gain.value = 1.0;
  }

  async load(id: string, url: string) {
    const response = await fetch(url);
    const arrayBuffer = await response.arrayBuffer();
    this.buffers.set(id, await this.ctx.decodeAudioData(arrayBuffer));
  }

  playSfx(id: string, options: { volume?: number; detune?: number } = {}) {
    const buffer = this.buffers.get(id);
    if (!buffer) return;

    // Resume context if suspended (browser autoplay policy)
    if (this.ctx.state === 'suspended') this.ctx.resume();

    const source = this.ctx.createBufferSource();
    const gain = this.ctx.createGain();
    source.buffer = buffer;
    source.detune.value = options.detune ?? 0;
    gain.gain.value = options.volume ?? 1;

    source.connect(gain).connect(this.sfxGain);
    source.start();
  }

  // Music crossfade — smooth transition between tracks
  async crossfadeTo(id: string, fadeDuration = 2) {
    const buffer = this.buffers.get(id);
    if (!buffer) return;
    if (this.ctx.state === 'suspended') await this.ctx.resume();

    const newSource = this.ctx.createBufferSource();
    newSource.buffer = buffer;
    newSource.loop = true;

    const newGain = this.ctx.createGain();
    newGain.gain.setValueAtTime(0, this.ctx.currentTime);
    newGain.gain.linearRampToValueAtTime(1, this.ctx.currentTime + fadeDuration);

    newSource.connect(newGain).connect(this.musicGain);
    newSource.start();

    if (this.currentMusic) {
      const oldGain = this.ctx.createGain();
      oldGain.gain.setValueAtTime(1, this.ctx.currentTime);
      oldGain.gain.linearRampToValueAtTime(0, this.ctx.currentTime + fadeDuration);
      this.currentMusic.connect(oldGain).connect(this.musicGain);
      const old = this.currentMusic;
      setTimeout(() => old.stop(), fadeDuration * 1000);
    }

    this.currentMusic = newSource;
  }

  // Spatial (3D) audio — attenuate by distance from listener
  playSpatial(id: string, x: number, y: number, z: number) {
    const buffer = this.buffers.get(id);
    if (!buffer) return;

    const source = this.ctx.createBufferSource();
    const panner = this.ctx.createPanner();
    source.buffer = buffer;

    panner.panningModel = 'HRTF';
    panner.distanceModel = 'inverse';
    panner.refDistance = 1;
    panner.maxDistance = 100;
    panner.rolloffFactor = 1;
    panner.positionX.value = x;
    panner.positionY.value = y;
    panner.positionZ.value = z;

    source.connect(panner).connect(this.sfxGain);
    source.start();
  }

  setMasterVolume(v: number) { this.masterGain.gain.value = Math.max(0, Math.min(1, v)); }
  setMusicVolume(v: number) { this.musicGain.gain.value = Math.max(0, Math.min(1, v)); }
  setSfxVolume(v: number) { this.sfxGain.gain.value = Math.max(0, Math.min(1, v)); }
}
```
