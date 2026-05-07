# Deep Creative Studio — Expansion Guide

## Vision
A gamified 3D space website. Each scroll takes the user to a planet. Each planet is a section (services, portfolio, about, contact). The journey through space IS the navigation. 3D is the atmosphere — HTML is the content.

---

## Stack Constraints (Non-Negotiable)
- No build tools, no npm, no bundler — plain HTML + CDN imports
- Three.js v0.166 via importmap
- All JS either inline `<script>` or `<script type="module">`
- Must run via `python3 -m http.server 8080` (GLB/WebM need a server)

---

## Workflow Rule — Index Is Always Latest

**`index.html` is always the current production version.** When a fundamental approach changes (new rendering method, new architecture), the old index becomes `index-vN.html` and the new version takes over as `index.html`. This is not always a merge — sometimes it's a full rebuild. Both paths are valid:

- **Incremental**: add features directly to `index.html`, backup + push at each milestone
- **Rebuild**: build a new version in a working file, validate it, then promote it to `index.html` and archive the previous as `index-vN.html`

Before every risky change: copy `index.html` → `backup/index.html`. Push to GitHub after each milestone.

`space-test.html` is the planet/travel prototype. When it's ready, it either gets merged into `index.html` or replaces it — whichever makes more sense at that point.

---

## Current Status (2026-05-07)

### What exists today

| File | Purpose | State |
|------|---------|-------|
| `index.html` | Main — GPU star field, GLSL comets, 3D logo, fly-through | ✅ Production-ready |
| `index-v1.html` | Archived — Canvas 2D black hole, vortex stars, phase 2 big bang | 🗄️ Archive |
| `space-test.html` | Prototype — 3 planets (Our Work, Services, Contact Us), 8 waypoints, seamless loop | 🔧 In progress |
| `EXPANDING.md` | This guide | ✅ |

### space-test.html — compliance check

| Rule | Status | Notes |
|------|--------|-------|
| One WebGL context | ✅ | Single renderer throughout |
| GPU stars (`THREE.Points`) | ✅ | 9,000 points, zero CPU |
| Procedural planets | ✅ | ShaderMaterial, no files |
| DOM layers for content | ✅ | Fixed divs fade in at arrival |
| Planets hidden on hero load | ✅ | `visible=false` until scroll > 1 |
| Canvas 2D particles reduced | ✅ | 1,000 stars (was 2,000) |
| Tab-hidden pause logic | ✅ | Both RAF loops covered |
| 60fps cap on RAF loops | ❌ | Phase 2 — not done yet |
| Proximity loading / disposal | ❌ | **Critical gap — see below** |
| `dispose()` when leaving | ❌ | GPU memory never freed |
| Antialias conditional | ⚠️ | Always on, should be device-aware |

### The critical gap before adding more 3D objects

Right now all planets load at page open and stay in GPU memory forever. With 2 small procedural planets this is fine. The moment real GLB objects are added per planet, it becomes a problem:

```
Current (broken) pattern:
Page load → ALL planets + ALL GLBs load → ALL stay in GPU forever

Target pattern:
Page load → hero GLB only
Approach planet 1 → planet 1 GLB loads in background
Leave planet 1 behind → planet 1 GLB disposed from GPU
```

**The proximity loading system must be built before any GLB is added to a planet.**
Without it: 5 planets × 300KB GLB = 1.5MB loaded on page open, all burning GPU memory simultaneously.

---

## Architecture: One Scene, Camera Uses Waypoints

All planets live in the **same Three.js scene** in completely different directions — not a straight line. Camera travels through waypoints defined as `{ pos, look }` pairs. Between waypoints, planets in other directions are invisible (outside FOV).

```
Hero (Z=0)
   │
   ▼  scroll 1→2: deep dive, straight forward, no planets visible
   │
   ▼  scroll 2→3: camera swings RIGHT → Planet 1 appears
   │
   ▼  scroll 3→4: camera swings FAR LEFT → Planet 2 (completely different direction)
   │
   ▼  scroll 4→5: camera returns RIGHT → Planet 1 again (journey loops)
```

Waypoint system (currently in space-test.html):
```js
const SPACE_WP = [
    { pos: Vector3(exit hero),   look: Vector3(straight ahead)   }, // sp=0
    { pos: Vector3(deep space),  look: Vector3(further ahead)    }, // sp=1 no planets visible
    { pos: Vector3(right side),  look: Vector3(planet 1 pos)     }, // sp=2 swing right
    { pos: Vector3(left side),   look: Vector3(planet 2 pos)     }, // sp=3 swing left
    { pos: Vector3(right again), look: Vector3(planet 1 pos)     }, // sp=4 return
];
```

**Key rule:** planets must be placed far enough in X/Y that they are outside the camera's FOV during the straight dive. With FOV=45° and planets placed at X=±38, they are outside the ~16-unit half-width at that depth. Always verify this when adding new planets.

---

## Layer Split (Critical for Performance)

| Layer | Technology | What lives here |
|-------|-----------|----------------|
| Background | `THREE.Points` + GLSL shader | 9k–20k GPU stars — zero CPU |
| 3D world | Three.js (existing renderer) | Logo, planets, travel path |
| Section content | Fixed DOM divs | Text, cards, forms — regular HTML |
| Hero effects | Canvas 2D (reduced) | Black hole particles on hero only |

**The 3D is the atmosphere. HTML is the content.**

---

## The Most Important Rule Before Adding 3D Objects: Proximity Loading + Disposal

Never load what the user hasn't reached. Never keep what they've passed.

```
User is at Planet 2:

[Planet 1]    [Planet 2]    [Planet 3]    [Planet 4]
  disposed     IN MEMORY     LOADING        queued
              (current)    (background)
```

Implementation pattern to build next:
```js
const LOAD_AHEAD    = 1.0; // start loading 1 section before arrival
const DISPOSE_BEHIND = 2.0; // dispose 2 sections behind current

// Each section is an object with load() and dispose() methods
const sections = [
    {
        loaded: false,
        load() {
            // build geometry, load GLB if needed, add to scene
            this.loaded = true;
        },
        dispose() {
            // geometry.dispose(), material.dispose(), texture.dispose()
            // scene.remove(mesh)
            this.loaded = false;
        }
    },
    // ...
];

function updateSections(scrollProgress) {
    sections.forEach((section, i) => {
        const dist = scrollProgress - i;
        if (dist > -LOAD_AHEAD   && !section.loaded) section.load();
        if (dist >  DISPOSE_BEHIND && section.loaded) section.dispose();
    });
}
// Call updateSections(scrollProgress) inside the animate loop
```

**Always call `dispose()` on Three.js objects you remove:**
```js
mesh.geometry.dispose();
mesh.material.dispose();
if (mesh.material.map) mesh.material.map.dispose();
scene.remove(mesh);
renderer.renderLists.dispose();
```
Without this, GPU memory fills permanently — invisible objects keep costing memory even when far away.

---

## Two Types of 3D Content — Different Loading Rules

### Procedural (zero load cost — always prefer this)
Built in JS instantly, no network request, no file:
- Planet body: `SphereGeometry` + `ShaderMaterial`
- Atmosphere: larger sphere, `side: THREE.BackSide`, additive blend
- Rings: `TorusGeometry`
- Background stars: `THREE.Points` with `BufferGeometry`

### Asset-based (lazy load only — never eager)
- Special 3D objects per section → GLB files, load 1 section ahead, dispose 2 sections behind
- Videos → create `<video>` element only when approaching, destroy when far away
- **Rule: if it can be procedural, it must be procedural**

---

## Planet Design (Current implementation in space-test.html)

Each planet = 4 meshes, all procedural, ~4 draw calls:
- Body: `SphereGeometry(r, 56, 56)` + `ShaderMaterial` (animated GLSL surface)
- Atmosphere: `SphereGeometry(r*1.20)` BackSide + additive blend, opacity 0.13
- Halo: `SphereGeometry(r*1.55)` BackSide + additive blend, opacity 0.06
- Ring: `TorusGeometry` × 2, DoubleSide, opacity 0.28 / 0.12

Planet groups set to `visible = false` on load — only shown when `scrollProgress > 1.0`.

Adding a GLB object to a planet (when ready):
```js
// Inside the section's load() method:
new GLTFLoader().load('assets/models/planet1-object.glb', gltf => {
    planetGroup.add(gltf.scene);
});
// Inside dispose():
planetGroup.remove(gltfScene);
gltfScene.traverse(c => { if (c.isMesh) { c.geometry.dispose(); c.material.dispose(); } });
```

---

## Camera Quaternion Reset on Return

When user scrolls back to hero from space, camera must reset both position AND rotation. Without the quaternion reset, the logo appears off-center. Current implementation in space-test.html:

```js
if (scrollProgress <= 1.0) {
    // X/Y snap back faster (0.10) so logo recenters quickly
    camera.position.x += (0 - camera.position.x) * 0.10;
    camera.position.y += (0 - camera.position.y) * 0.10;
    camera.position.z += (tZ - camera.position.z) * 0.06;
    // Quaternion slerp back to looking at origin
    _tmpCam.lookAt(0, 0, 0);
    camera.quaternion.slerp(_tmpCam.quaternion, 0.10);
}
```

---

## Background Stars: GPU Points

`THREE.Points` — all star rendering on GPU, zero per-frame CPU:

```js
const count = 9000; // current — can go up to 20k safely
const pos = new Float32Array(count * 3);
const col = new Float32Array(count * 3);
// fill positions (wide volume covering travel path) + per-star colour
const mat = new THREE.PointsMaterial({
    size: 0.55, sizeAttenuation: true,
    vertexColors: true, transparent: true, opacity: 0.82, depthWrite: false
});
scene.add(new THREE.Points(geo, mat));
```

Keep Canvas 2D only for the black hole physics (interactive, cannot move to GPU without compute shaders).

---

## Section Content Pattern (DOM Layers)

```html
<div id="planet-1-layer" class="planet-layer" aria-hidden="true">
    <p class="pl-eyebrow">Planet 01</p>
    <h2 class="pl-title">Section Title</h2>
    <p class="pl-sub">Description text.</p>
    <button class="pl-cta">Call to Action</button>
</div>
```
```css
.planet-layer {
    position: fixed; inset: 0; z-index: 20;
    opacity: 0; pointer-events: none;
    transition: opacity 0.8s ease;
    display: flex; flex-direction: column;
    align-items: center; justify-content: center;
}
.planet-layer.visible { opacity: 1; pointer-events: auto; }
```
Toggled by `classList.toggle('visible', arrived)` in the animate loop.

---

## Initial Page Load Budget (Target)

```
logo_3d_optimized.glb    ~297KB   (hero — eager load)
Three.js CDN             ~580KB   (cached after first visit)
HTML + inline JS         ~80KB
Fonts (Google CDN)       ~20KB
─────────────────────────────────
Total first load         ~977KB   ≈ 1MB   ✅ on target
```
Everything else (planet GLBs, videos) loads on-demand via proximity system.

---

## Golden Rules — Must Follow Before Every Feature

### 1. One WebGL Context
Never add a second `new THREE.WebGLRenderer()`. One renderer, move the camera.

### 2. Cap to 60fps (PENDING — Phase 2)
```js
let _lastFrame = 0;
function animate(now) {
    requestAnimationFrame(animate);
    if (now - _lastFrame < 16.67) return;
    _lastFrame = now;
}
animate(0);
```

### 3. Pause When Tab Hidden
```js
document.addEventListener('visibilitychange', () => {
    _running = !document.hidden;
    if (!document.hidden) animate();
});
```

### 4. Dispose Everything You Remove
```js
mesh.geometry.dispose();
mesh.material.dispose();
if (mesh.material.map) mesh.material.map.dispose();
scene.remove(mesh);
```

### 5. Prefer Procedural Over Files
If geometry + shader can do it, don't load a file.

### 6. Never Eager-Load Planet Assets
Planet GLBs and videos load only when 1 section away. The proximity system is the gatekeeper.

---

## Asset Guidelines

| Asset type | Max size | Format | Loading rule |
|-----------|---------|--------|-------------|
| Hero logo | 500KB | GLB (Draco) | Eager — page load |
| Planet body | 0KB | Procedural ShaderMaterial | Instant |
| Planet special object | 300KB max | GLB (Draco) | Lazy — 1 section ahead, disposed 2 behind |
| Section video | 2MB max | WebM (VP9) | Lazy — 1 section ahead, paused when invisible |
| Textures | 512×512 | WebP | Bake in Blender, lazy load |
| Fonts | CDN | Google Fonts | Eager — `<head>` |

---

## Performance Budget

| Metric | Target | index.html | space-test.html |
|--------|--------|-----------|----------------|
| WebGL contexts | 1 | ✅ 1 | ✅ 1 |
| RAF loops | ≤ 3 | ✅ 3 | ✅ 3 |
| 60fps cap | required | ❌ pending | ❌ pending |
| Canvas 2D particles | ≤ 1000 | ✅ 1000 | ✅ 1000 |
| GPU stars | THREE.Points | ✅ 6k points | ✅ 9k points |
| Proximity loading | required | n/a | ❌ not built |
| dispose() on removal | required | n/a | ❌ not built |
| Initial load | < 1.5MB | ✅ ~1MB | ✅ ~1MB |
| Chrome GPU CPU | < 15% | ⚠️ measuring | ⚠️ measuring |

---

## Build Order — What Must Happen Before Adding 3D Objects

**Phase 2 — 60fps cap** (quick, should be next)
Add delta-time guard to all 3 RAF loops.

**Phase 3 — Cache `_camScrollZ`**
Pass it into VortexStar instead of reading `window._camScrollZ` 1000× per frame.

**Phase 4 — Debounce resize**
Prevent 1000 particle reinits when dragging window edge.

**Phase 5 — Proximity loading system** ← MUST BE DONE BEFORE ANY PLANET GLB
Convert planets array to section objects with `load()` / `dispose()` methods.

**Phase 6 — First real planet GLB**
Only after Phase 5. One GLB, lazy-loaded, properly disposed.

**Phase 7 — Remaining planets**
Each one follows the same load/dispose pattern.

**Phase 8 — Polish**
Arrival effects, travel trail particles, transitions.

---

## File Structure

```
deep-creative/
  index.html              — ALWAYS the current production version
  index-v1.html           — archived: Canvas 2D black hole, vortex stars, big bang
  space-test.html         — planet/travel prototype (3 planets, 8 waypoints, seamless loop)
  logo_3d_optimized.glb   — hero logo (eager)
  cloud_pulse.webm        — background video
  assets/
    models/               — planet GLBs (lazy, 300KB max each, Draco compressed)
    videos/               — section videos (lazy, 2MB max each, WebM VP9)
  backup/
    index.html            — update before every risky change
  CLAUDE.md               — project diary
  EXPANDING.md            — this file
```

When `space-test.html` is ready to become production, either merge its systems into `index.html` or promote it directly — whichever is cleaner at that point. Archive the old `index.html` as `index-v2.html`.

If the codebase grows large enough to warrant splitting:
- `js/particles.js` — Canvas 2D hero effects
- `js/scene.js` — Three.js scene, logo, camera
- `js/scroll.js` — scroll/navigation/waypoints
- `js/stars.js` — GPU star field
- `js/sections.js` — planet definitions + load/dispose system

---

*Last updated: 2026-05-07 — reflects GPU star rebuild (v1→current index), space-test 3-planet prototype*
