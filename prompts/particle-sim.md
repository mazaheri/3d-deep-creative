# WebGL Particle Swarm Simulator Prompt

Use this prompt with an AI to generate particle behavior code for a 20,000+ unit 3D swarm simulator.

## API

| Variable | Type | Notes |
|----------|------|-------|
| `i` | Integer | Current particle index (0 to count-1) |
| `count` | Integer | Total particles |
| `target` | THREE.Vector3 | **Write-only** — set position via `target.set(x,y,z)` |
| `color` | THREE.Color | **Write-only** — set via `color.setHSL(h,s,l)` or `color.set(...)` |
| `time` | Float | Global time in seconds |
| `THREE` | Object | Full Three.js access |

## Helper Functions

```js
addControl("id", "Label", min, max, initialValue) // returns float — creates UI slider
setInfo("title", "description")                   // call only when i === 0
annotate("id", new THREE.Vector3(x,y,z), "label") // call only when i === 0
```

## Rules (enforced by validator)

- No `new THREE.Vector3()` / `new THREE.Color()` inside loop — zero GC
- No `if/else` branching — use math (`Math.sin`, `Math.abs`, etc.)
- No return value — mutate `target` and `color` only
- All values must be finite (guard divisions against zero)
- Forbidden: `document`, `window`, `fetch`, `eval`, `Function(`, `import(`, `require(`, `process`, `__proto__`, `.prototype`, `globalThis`, `self`, `localStorage`, `setTimeout`, `setInterval`
- Declare all variables with `let` or `const`

## Prompt Template

```
Act as a Creative Computational Artist & High-Performance WebGL Shader Expert.

Write a single optimized JavaScript function body for a 20,000+ particle 3D swarm simulation.

API: i (index), count (total), target (THREE.Vector3, write-only), color (THREE.Color, write-only), time (float), THREE (library).
Helpers: addControl(id, label, min, max, init), setInfo(title, desc) [i===0 only], annotate(id, vec3, label) [i===0 only].

Rules: no new objects in loop, no return, all values finite, no forbidden globals, declare all vars.

REQUEST: [YOUR IDEA HERE]

Return ONLY the JS code — no markdown, no backticks, no explanation.
```

## Example Output

```js
const scale = addControl("scale", "Expansion", 10, 100, 50);
const angle = i * 0.1 + time;
target.set(Math.cos(angle) * scale, Math.sin(angle) * scale, i * 0.05);
color.setHSL(i / count, 1.0, 0.5);
if (i === 0) setInfo("Spiral Demo", "A basic test.");
```

## Ideas for DCS Project

- Big Bang spiral galaxy birth
- Black hole accretion disk
- Lorenz attractor field
- Interference wave patterns
- Breathing tesseract / 4D shape
