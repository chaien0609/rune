---
name: "webgl"
pack: "@rune/gamedev"
description: "WebGL patterns — shader programming, GLSL, buffer management, texture handling, instanced rendering."
model: sonnet
tools: [Read, Edit, Write, Grep, Glob, Bash]
---

# webgl

WebGL patterns — shader programming, GLSL, buffer management, texture handling, instanced rendering.

#### Workflow

**Step 1 — Detect WebGL usage**
Use Grep to find WebGL code: `getContext('webgl`, `gl.createShader`, `gl.createProgram`, `*.glsl`, `*.vert`, `*.frag`. Read shader files and GL initialization to understand: WebGL version, shader complexity, and buffer strategy.

**Step 2 — Audit shader and buffer efficiency**
Check for: uniforms set every frame that don't change (use UBO), separate draw calls for identical geometry (use instancing), textures not using mipmaps, missing `gl.deleteBuffer`/`gl.deleteTexture` cleanup, and shaders with expensive per-fragment branching.

**Step 3 — Emit optimized WebGL code**
Emit: WebGL2 setup with proper context attributes, VAO-based buffer management, instanced rendering for repeated geometry, and GLSL shaders with documented inputs/outputs.

#### Example

```glsl
// Vertex shader — instanced rendering with per-instance transform
#version 300 es
layout(location = 0) in vec3 aPosition;
layout(location = 1) in vec3 aNormal;
layout(location = 2) in mat4 aInstanceMatrix; // per-instance (locations 2-5)

uniform mat4 uViewProjection;

out vec3 vNormal;
out vec3 vWorldPos;

void main() {
  vec4 worldPos = aInstanceMatrix * vec4(aPosition, 1.0);
  vWorldPos = worldPos.xyz;
  vNormal = mat3(transpose(inverse(aInstanceMatrix))) * aNormal;
  gl_Position = uViewProjection * worldPos;
}
```

```glsl
// Fragment shader — PBR-lite with single directional light
#version 300 es
precision highp float;

in vec3 vNormal;
in vec3 vWorldPos;
out vec4 fragColor;

uniform vec3 uLightDir;
uniform vec3 uCameraPos;
uniform vec3 uBaseColor;

void main() {
  vec3 N = normalize(vNormal);
  vec3 L = normalize(uLightDir);
  vec3 V = normalize(uCameraPos - vWorldPos);
  vec3 H = normalize(L + V);

  float diffuse = max(dot(N, L), 0.0);
  float specular = pow(max(dot(N, H), 0.0), 32.0);
  vec3 ambient = uBaseColor * 0.15;

  fragColor = vec4(ambient + uBaseColor * diffuse + vec3(specular * 0.5), 1.0);
}
```
