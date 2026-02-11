# Pharia Brain Visualization — Developer Handoff

## Overview

We've built an interactive 3D brain visualization that shows how TMS therapy targets specific brain networks. It's currently a working prototype — your job is to integrate the 3D brain component into the real site across multiple pages/sections.

**Live demo:** https://drjeeshan.github.io/pharia-brain/v7-carousel.html  
**Repo:** https://github.com/drjeeshan/pharia-brain  
**Key file:** `v7-carousel.html` (everything is in this one file)

---

## What You're Looking At in V7

V7 is a full prototype page that includes several sections (hero, mental load, pricing, brain visualization, TMS info, CTA). **The part you need to extract is the interactive 3D brain section** — specifically the "The Science" section with the rotating brain and carousel of network cards.

The rest of the page (hero video, pricing cards, etc.) is placeholder layout that your site will handle natively.

---

## What the Brain Component Does

1. Loads a 3D brain model (`brain.glb`, ~12MB, 283 meshes)
2. Renders it with a translucent wireframe aesthetic and glow effects
3. Users can rotate/zoom the brain with mouse/touch
4. Four brain networks can be highlighted — when one is selected, those brain regions glow in their assigned color while everything else fades to near-transparent
5. A carousel of cards lets users switch between the four networks
6. Each card has a "Learn More" button that opens a YouTube video modal

---

## Files You Need From the Repo

| File | Purpose |
|------|---------|
| `v7-carousel.html` | All source code (JS, CSS, HTML in one file) — extract from here |
| `brain.glb` | The 3D brain model — must be served as a static asset |
| `og-image.png` | Social sharing preview image (1200×630) |

---

## Extracting Into a Next.js / React Component

Everything you need is in the `<script type="module">` block at the bottom of `v7-carousel.html`. Here's what maps to what:

### Three.js Dependencies (install via npm)

```bash
npm install three
```

All imports at the top of the script block map to:
```js
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
import { ShaderPass } from 'three/addons/postprocessing/ShaderPass.js';
import { GammaCorrectionShader } from 'three/addons/shaders/GammaCorrectionShader.js';
```

### Key Architecture Pieces to Extract

**1. The `networks` object** — This is the data model. It defines the four networks with their colors, descriptions, mesh names, and video links. Pull this out as a config/data file.

**2. Scene initialization** — The renderer, camera, composer (bloom + gamma correction), orbit controls, and lighting setup. This becomes the core of your React component's `useEffect`.

**3. `highlightNetwork(networkId)`** — The function that makes a selected network glow and fades everything else. This is the main interaction logic.

**4. Model loading** — The GLTFLoader that loads `brain.glb`, traverses meshes, and creates both solid and wireframe materials for each mesh.

**5. Animation loop** — The `animate()` function that handles auto-rotation, pulse effect, and rendering.

**6. Carousel + Video Modal** — These are the UI layer. You'll likely rebuild these in React rather than extracting the vanilla JS.

### Suggested React Component Structure

```
BrainVisualization/
├── BrainScene.tsx          // Three.js canvas, scene, renderer, model loading
├── NetworkCarousel.tsx     // Card carousel UI
├── VideoModal.tsx          // YouTube embed modal
├── networks.ts             // Network data (colors, descriptions, mesh names, videos)
└── useBrainScene.ts        // Hook: Three.js lifecycle, cleanup, resize handling
```

The Three.js scene should use a `ref` for the canvas element, initialize in `useEffect`, and expose a `highlightNetwork(id)` method that the carousel calls when the user switches cards.

---

## Critical Technical Requirements

These were discovered through debugging and are easy to get wrong:

### 1. Color Management Must Be Disabled
```js
THREE.ColorManagement.enabled = false;
```
Set this *before* creating any materials. Without it, Three.js applies color space conversions that shift the network colors.

### 2. Specific Color Space + Tone Mapping
```js
renderer.outputColorSpace = THREE.LinearSRGBColorSpace;
renderer.toneMapping = THREE.NoToneMapping;
```

### 3. Post-Processing Order Matters
```
RenderPass → UnrealBloomPass → GammaCorrectionShader
```
Don't use `OutputPass` — it causes a background color mismatch between the canvas and the page.

### 4. Dark Background Required
The bloom/glow effects only look right against a dark background. The current value is `#202621` on the page and `0x030403` for the Three.js scene background. If your page section has a light background, the brain viewer needs its own dark container.

### 5. Bloom Parameters (Designer-Approved)
```js
new UnrealBloomPass(resolution, 0.18, 0.12, 0.5)
// strength: 0.18, radius: 0.12, threshold: 0.5
```

### 6. Mobile Performance
- Pixel ratio capped at 2x: `renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))`
- Animation throttled to ~60fps
- Orbit controls have zoom limits: `minDistance: 1.2, maxDistance: 3`

---

## The Four Networks (Data Reference)

| Network ID | Display Name | Nickname | Color | TMS Target |
|-----------|-------------|----------|-------|-----------|
| `left-cen` | Proactive Control Network | "The CEO" | `#A4C8FF` | Left DLPFC |
| `right-cen` | Reactive Control Network | "The Filter" | `#8AB3F9` | Right DLPFC |
| `salience` | Salience Network | "The Priority Switch" | `#FFB974` | dmPFC |
| `dmn` | Default Mode Network | "The Inner Critic" | `#C8D4C8` | dmPFC |

### Mesh Mapping
Each network highlights specific named meshes in the GLB model. The mesh names follow the pattern `Allen_[structure]_[L/R]`. The full mapping is in the `networks` object in the script — search for `meshNames` in `v7-carousel.html`.

### Visual Behavior
- **Default state:** All meshes at 0.05 opacity with 0.05 wireframe opacity (near-invisible ghost)
- **Highlighted state:** Selected network meshes jump to 0.9 opacity with their color + emissive glow, plus a subtle pulse animation
- **Non-selected meshes** stay at 0.05 opacity so the highlighted deeper structures show through ("adaptive transparency")

---

## Educational Videos

Each network links to a YouTube clip from a neuroscience expert:

| Network | Expert | Video ID | Start Time (seconds) |
|---------|--------|----------|---------------------|
| left-cen | Dr. Nolan Williams | `X4QE6t-MkYE` | 2749 |
| right-cen | Dr. Jud Brewer | `1M-bUw-OiG0` | 1596 |
| salience | Dr. Nolan Williams | `X4QE6t-MkYE` | 2916 |
| dmn | Dr. Andrew Huberman | `wTBSGgbIvsY` | 1380 |

Embed URL pattern:
```
https://www.youtube.com/embed/{videoId}?autoplay=1&start={startTime}&rel=0&playsinline=1
```

---

## Serving the Brain Model

`brain.glb` is ~12MB. In the current prototype it loads from:
```
https://drjeeshan.github.io/pharia-brain/brain.glb
```

For production, copy `brain.glb` into your Next.js `public/` directory and update the loader path:
```js
loader.load('/brain.glb', ...)
```

Consider showing a loading state — the prototype uses a spinner overlay that fades out once the model is loaded.

---

## Using the Brain Across Multiple Pages

Since this will appear as a section on different pages, build it as a reusable component that accepts props:

```tsx
<BrainVisualization 
  initialNetwork="left-cen"    // Which network to highlight on load
  showCarousel={true}           // Whether to show the card carousel
  height="45vh"                 // Container height
/>
```

This way you can use the same component in different contexts:
- Full interactive version with carousel on the Science page
- Simplified version (single highlighted network, no carousel) on other pages
- Different initial network highlighted depending on page context

---

## Brand Fonts Referenced in V7

The prototype uses two custom fonts. Check with design whether the production site uses these or different ones:

- **Founders Grotesk** (headlines) — loaded from `Brand Fonts/FoundersGrotesk Web/`
- **Matter** (body text) — loaded from `Brand Fonts/Matter Web/`

These font files are NOT in the GitHub repo. The carousel cards and UI text use these fonts, but the 3D brain itself doesn't depend on them.
