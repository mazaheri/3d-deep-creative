# Deep Creative Studio — 3D Website

## Project Overview
A single-page 3D landing page for Deep Creative Studio (DCS). No React, no build tools — plain HTML + Three.js via CDN importmap. Runs on a local Python server for development.

## Tech Stack
- **Three.js** (v0.166) via CDN importmap — 3D logo rendering
- **GLTFLoader** — loads `logo_3d_optimized.glb`
- **RoomEnvironment** — subtle env map for glass reflections
- **Canvas 2D API** — star particle system (separate canvas layer)
- **HTML5 Video** — `cloud_pulse.webm` background vortex effect
- **Tailwind CSS** via CDN

## Running Locally
```bash
cd "/Users/zattoo/Documents/Claude ai/projects/Deepstudio/3d website/deep-creative"
python3 -m http.server 8080
# Open http://localhost:8080
```
(Server needed because GLB and WebM can't load from file://)

## File Structure
```
deep-creative/
  index.html              — entire app (HTML + CSS + JS, all in one file)
  logo_3d_optimized.glb   — 3D logo model (~297KB, exported from Blender)
  cloud_pulse.webm        — background cloud/vortex video (~1.3MB, AI generated)
  CLAUDE.md               — this file (project diary)
  backup/
    index.html            — last known good state (update before risky changes)
```

## Canvas Layer Order (z-index)
| Layer | Element | z-index | Notes |
|-------|---------|---------|-------|
| 0 | `#vortex-canvas` (video) | 0 | Background cloud pulse, screen blend |
| 1 | `#three-canvas` | 1 | Three.js 3D logo |
| 2 | `#particleCanvas` | 3 | Stars, comets, shooting stars |
| 3 | `#ui-layer` | 10 | Hint text |

## What's Built So Far

### 3D Logo (Three.js)
- Loads `logo_3d_optimized.glb` — DCS logo made of ring meshes + sparkles
- Glass material: dark blue, metalness 0.4, roughness 0.05, emissive `#26b2db`
- Lights: key, fill, two rims, scan beam, pulse flash, mouse proximity light
- **Hover**: proximity drives light boost + color shift to `#76e0fa`, camera zooms in slightly
- **Scroll fly-through**: camera flies from z=3.5 to z=-2.5 (6 world units), rings spread and fade as camera passes through
- **Scroll light-up**: logo lights up (proximity → 1) the moment scroll starts, before fly-through begins

### Star Particle System (Canvas 2D)
- 2000 `VortexStar` — orbit center in a tilted disk (DISK_TILT 0.85), twinkle
- 16 `ShootingStar` — streak across the scene
- 5 `Comet` — slow drift, ignite on mouse proximity or click, streak toward center
- Stars zoom in sync with camera scroll via `window._camScrollZ`

### Black Hole Mechanics
- `innerR = min(w,h) * 0.09` — event horizon (logo center)
- `GRAV_ZONE = min(w,h) * 0.16` — catch boundary (expands during scroll)
- `MAGNET_ZONE = GRAV_ZONE * 1.7` — pre-catch attraction zone
- `OUTER_DRAG_ZONE = GRAV_ZONE * 2.8` — drag felt by escapees beyond magnet zone

**Star fate system:**
- Each star has a `resistance` (0–1) assigned at birth
- Low resistance (< 0.58): pulled in smoothly, spirals inward (Keplerian speedup)
- High resistance (> 0.58): fights back, escapes outward with burst — but:
  - Magnetic pull opposes escape (closer = harder to flee)
  - Each escape reduces resistance by 0.20
  - After 2–3 encounters, even stubborn stars get consumed
  - Escapees feel drag all the way to OUTER_DRAG_ZONE (slow, gravitational return)
- Caught stars: shrink + spaghettify (stretch toward center) as they spiral in
- On consumption: absorption flash (blue-white glow), new star spawns from outer edge immediately
- During scroll: gravity zone expands, tidal pull on all free stars, spiral slows (visible longer)

### Logo Mask
Before scroll: stars are erased from the logo area using `destination-out` compositing — logo stays clean and visible. Mask fades out in the first 25% of scroll progress.

### Background Video
- `cloud_pulse.webm` fades in after 3s delay, loops with 4.5s gap between loops
- Fades out when hovering over logo or scrolling
- `mix-blend-mode: screen` so black areas are transparent

## Git Remote
https://github.com/mazaheri/3d-deep-creative.git

## Current State (last updated: 2026-05-06)
- ✅ Star galaxy with black hole eat mechanic
- ✅ Mouse interaction (push stars toward hole)
- ✅ Resistance / escape / weakening system
- ✅ Scroll fly-through with logo light-up
- ✅ Logo mask (stars hidden under logo before scroll)
- ✅ Background cloud pulse video
- ✅ Pushed to GitHub

## Next — Ideas to Explore
- Second slide / section after fly-through
- More GLB objects in other slides
- Possible: shader-based cloud effect to replace video
- Possible: post-processing (bloom, depth of field) on Three.js canvas

## How to Update This Diary
Before starting a session: read this file.
After a session: update "Current State" and "Next" sections with what changed.
When making risky changes: copy `index.html` → `backup/index.html` first.
