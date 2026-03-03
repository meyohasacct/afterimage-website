# Molecule Poetry — Build Log

## Overview

`experiments/molecule-poetry.html` is an interactive, scroll-driven web experience for **AFTERIMAGE by Studio Meyohas**. It combines a 2D canvas-based typographic animation with a 3D perfume bottle rendered in Three.js, all in a single self-contained HTML file.

**Live URL:** https://meyohasacct.github.io/afterimage-website/experiments/molecule-poetry.html

---

## Architecture

### Dual-Canvas System
The page uses two overlapping full-viewport canvases:

1. **2D Canvas (`#c`)** — Renders the "molecular poetry" text animation: scattered letters that converge into the AFTERIMAGE logotype as the user scrolls. Built with vanilla Canvas 2D API + simplex noise for organic movement.

2. **Three.js Canvas (`#three-canvas`)** — Renders the 3D perfume bottle (glass body + pump) with real-time lighting and reflections. Overlaid with `pointer-events: none` and transparent background so the 2D canvas shows through.

### Scroll Mapping
The page is scroll-driven with no click interactions. Total page height:
- **Desktop:** 1100vh
- **Mobile (≤768px):** 600vh

Scroll is divided into three zones:

| Zone | Desktop | Mobile | What Happens |
|------|---------|--------|--------------|
| **Letter animation** | 0 → 4 screens | 0 → 2.5 screens | Scattered letters converge into "AFTERIMAGE" |
| **Hold zone** | 3 screens | 1 screen | Logotype + byline stay visible for reading |
| **Bottle reveal** | Remaining scroll | Remaining scroll | Text slides up, 3D bottle fades in with "COMING LATE 2026" |

Key variables:
- `scrollT` (0→1): Controls letter dispersion → alignment
- `bottleT` (0→1): Controls bottle fade-in and caption visibility

---

## 2D Text Animation

### Simplex Noise
A seeded simplex noise generator (seed: 42) drives organic letter movement. Each letter has independent noise-based vibration that diminishes as letters align.

### 3D Perspective
Letters exist in 3D space and are projected to 2D with a perspective camera (`PERSPECTIVE = 800`). Rotation around Y and X axes adds depth. As `scrollT` increases, rotation eases out and letters flatten to their final aligned positions.

### Formations
- **Dispersed:** Random 3D positions within a sphere, re-randomized every 3 seconds when near the top
- **Centered Word:** Letters spaced evenly on a horizontal line at the origin

Spring physics (`SPRING_K = 1.3`, `DAMPING = 0.83`) smoothly interpolate between formations.

### Molecular Bonds
Faint lines connect nearby letters (proximity-based when dispersed, chain-based when aligned), creating a molecular/chemical structure aesthetic. Bonds fade with scroll progress.

### Byline
"BY STUDIO" and "MEYOHAS" appear below the logotype at `scrollT > 0.9`, drawn on the 2D canvas in the Kalice font. Spacing: MEYOHAS is offset `bylineSize * 1.15` below BY STUDIO.

### Text Slide-Up
When the bottle begins appearing (`bottleT > 0`), the entire text block (logotype + byline) slides upward to `12vh` from the top of the viewport, mirroring the caption position at the bottom.

---

## 3D Bottle (Three.js)

### Technology Stack
- **Three.js v0.162.0** via ES module importmap
- **GLTFLoader** + **DRACOLoader** for compressed 3D models
- **RoomEnvironment** + **PMREMGenerator** for procedural studio cubemap

### Model Files
Two separate GLB files in `experiments/`:

| File | Original | Final | Reduction |
|------|----------|-------|-----------|
| `bottle-body.glb` | 15 MB GLTF (416K vertices) | 274 KB GLB (~42K vertices) | 98% |
| `bottle-pump.glb` | 2.1 MB GLTF | 72 KB GLB | 97% |

Compression pipeline: mesh simplification via `gltf-transform` (reduced to ~10% of triangles) → Draco compression → GLB encoding.

### Glass Material
The glass body uses `MeshPhysicalMaterial` **without** transmission (transmission was tested extensively but caused unacceptable lag due to extra render passes and looked gray/black on dark backgrounds):

```javascript
MeshPhysicalMaterial({
  color: 0x111111,
  metalness: 0.0,
  roughness: 0.05,
  envMap: envMap,          // RoomEnvironment cubemap
  envMapIntensity: 2.5,
  clearcoat: 1.0,
  clearcoatRoughness: 0.0,
  ior: 1.5,
  reflectivity: 1.0,
  transparent: true,
  opacity: 0.30,
  side: DoubleSide,
  depthWrite: false,       // So pump is visible through glass
})
```

Key insight: On dark backgrounds, flat glass faces need **environment map reflections** (not just Fresnel edge glow) to look glossy. The `clearcoat` layer adds a second specular highlight pass that sells the glass effect. `depthWrite: false` + `renderOrder` (pump=0, glass=1) ensures the pump is visible through the transparent body.

### Pump Material
```javascript
MeshStandardMaterial({
  color: 0x8a0600,    // Dimmed rose red (brand palette)
  metalness: 0.8,
  roughness: 0.15,
})
```

### Lighting Setup
- **Ambient light:** 0.3 intensity base fill
- **Key light:** Directional, intensity 3, from upper-right-front
- **Fill light:** Directional, intensity 2, from left
- **Back light:** Directional, intensity 2.5, from behind
- **Top light:** Directional, intensity 1.5, from above
- **Red side spotlight:** SpotLight with color `0xC60900`, intensity 12, narrow cone (`π/6`), high penumbra (0.7)
- **Orbiting white spotlight:** SpotLight intensity 25, orbits the bottle at radius 5.6 units, completing a full 360° rotation every 5 seconds. Starts from the camera position (z+).

### Renderer Settings
- `ACESFilmicToneMapping` with `toneMappingExposure = 2.5`
- Pixel ratio capped at 2 for performance
- Alpha-enabled (transparent background)

### Bottle Animation
- Slow continuous Y-axis rotation (`+= 0.004` per frame)
- Fades in during `bottleT 0 → 0.3`
- Warm-up render during loading screen pre-compiles shaders

---

## Loading Screen

A "LOADING" text with per-letter wave animation (80ms stagger between letters, 1-second animation cycle). Each letter oscillates between 0.2 and 1.0 opacity. The loader blocks scrolling (`html.loading` sets `height: 100vh; overflow: hidden`) until both 3D models are loaded and the font is ready.

Dismiss: `opacity` transition over 0.6 seconds, then removed from DOM.

---

## "COMING LATE 2026" Caption

Positioned fixed at `bottom: 12vh`, centered, using the Kalice font. Fades in with the bottle (`bottleT` controls opacity). Single line of text, all caps, with `0.2em` letter-spacing.

---

## Mobile Responsiveness (≤768px)

| Property | Desktop | Mobile |
|----------|---------|--------|
| Page height | 1100vh | 600vh |
| Letter scroll range | 4 screens | 2.5 screens |
| Hold zone | 3 screens | 1 screen |
| Canvas font size ratio | 0.045 | 0.09 (2x) |
| Caption font size | clamp(12px, 1.4vw, 18px) | clamp(24px, 2.8vw, 36px) |
| Loader font size | 14px | 28px |
| Grain overlay | 8% opacity | Hidden |
| Scroll restoration | N/A | Disabled (`history.scrollRestoration = 'manual'`) |

The grain overlay (animated SVG fractal noise) is completely hidden on mobile to reduce visual noise and improve performance.

Scroll restoration is disabled via `history.scrollRestoration = 'manual'` + `window.scrollTo(0, 0)` to prevent the browser from restoring a mid-page scroll position on reload, which would show the bottle immediately without the scroll journey.

---

## Custom Font

The page uses **Kalice** (400 weight with curly R variant) loaded from `../assets/fonts/Kalice-Regular-CurlyR.woff2`. The curly R is a distinctive brand element. A medium weight (500) is also declared but the primary display uses 400.

Important: The font file must be served from the website root — running a dev server from the `experiments/` subdirectory will break the relative path.

---

## Performance Optimizations

1. **Mesh simplification:** 416K → ~42K vertices via gltf-transform, 98% file size reduction
2. **Draco compression:** Further reduces GLB transfer size
3. **No transmission:** MeshPhysicalMaterial with envMap instead of transmission avoids expensive extra render passes
4. **Warm-up render:** Two render passes during loading screen pre-compile shaders so first visible frame isn't janky
5. **Conditional rendering:** Three.js only renders when `bottleT > 0` (bottle is in view)
6. **Pixel ratio cap:** `Math.min(devicePixelRatio, 2)` prevents excessive resolution on high-DPI mobile
7. **No grain on mobile:** Eliminates animated SVG noise overlay on smaller devices

---

## Glass Material Evolution

The glass material went through ~7 iterations before reaching the final approach:

1. **MeshPhysicalMaterial + transmission** → Looked gray (nothing behind glass to refract on dark background), extremely laggy (extra render pass)
2. **Transparent MeshPhysicalMaterial** → Better performance but no visual interest on flat faces
3. **Custom Fresnel ShaderMaterial** → Only edges glowed, flat faces looked matte/ghostly
4. **Fresnel + fake studio environment reflections** → Improved but still matte
5. **Fresnel + specular highlights** → Still not convincing
6. **Transmission attempt #2** (inspired by Olivier Larose demo) → Black and laggy
7. **Final: MeshPhysicalMaterial + envMap + clearcoat (no transmission)** → Glossy reflections from RoomEnvironment cubemap, clearcoat adds second specular layer, performant

The key lesson: for glass on dark backgrounds, you need **environment reflections on flat faces** (via envMap), not just edge effects (Fresnel). The clearcoat layer adds the glossy "wet" look without the performance cost of transmission.

---

## File Structure

```
website/
├── experiments/
│   ├── molecule-poetry.html    # The complete experience (single file)
│   ├── bottle-body.glb         # Glass body (274 KB, Draco-compressed)
│   └── bottle-pump.glb         # Pump mechanism (72 KB, Draco-compressed)
├── assets/
│   └── fonts/
│       ├── Kalice-Regular-CurlyR.woff2
│       └── Kalice-Medium.woff2
└── MOLECULE-POETRY-BUILD-LOG.md
```

---

## Git History

```
22c99a2 Simplify COMING LATE 2026 to single line, adjust byline spacing
fa867ba Disable grain overlay on mobile
33124d4 Double mobile font sizes (2x desktop)
69f3b01 Increase mobile font sizes to 50% larger than desktop
8ed6e7c Fix mobile: prevent scroll restoration showing bottle on load, reduce grain
daeefa3 Increase font size by 20% on mobile for molecule poetry page
8932e97 Fix curly R font on GitHub Pages and mobile scroll speed
ccba84e Add 3D bottle to molecule poetry experiment
0793909 Remove source assets and screenshots from tracking
d4b102d Add .gitignore, remove source assets from tracking
4e98554 Initial commit: AFTERIMAGE pre-launch website
```

---

## Dependencies (all loaded via CDN)

- **Three.js v0.162.0** — 3D rendering engine
  - GLTFLoader — loads .glb models
  - DRACOLoader — decompresses Draco-encoded meshes (decoder from Three.js CDN)
  - RoomEnvironment — procedural studio lighting cubemap for reflections
  - PMREMGenerator — pre-filters environment map for PBR materials
- **SimplexNoise** — Inline implementation (no external dependency), seeded for deterministic organic motion
