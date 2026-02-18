# Pharia Brain Visualization — Developer Handoff

## Overview

We've built an interactive 3D brain visualization that shows how TMS therapy targets specific brain networks. It's currently a working prototype — your job is to integrate the 3D brain component into the real site across multiple pages/sections.

**Live demo:** https://drjeeshan.github.io/pharia-brain/v7-carousel.html  
**Repo:** https://github.com/drjeeshan/pharia-brain  
**Key file:** `v7-carousel.html` (everything is in this one file)

---

## What You're Looking At in V7

V7 is a full prototype page that includes several sections (hero, mental load, pricing, brain visualization, TMS info, CTA). **The part you need to extract is the brain visualization section** — specifically:

1. **The 3D brain viewer** — a Three.js scene rendering a translucent brain with highlighted network regions
2. **The network carousel** — cards below the brain that let users switch between four networks
3. **The video modal** — YouTube embeds that open when users click "Learn more"

Everything else on the page (hero, pricing, etc.) is prototype layout that your Figma/design will replace.

---

## Files You Need

| File | What It Is | Size |
|------|-----------|------|
| `v7-carousel.html` | Complete application (single HTML file with embedded CSS + JS) | ~64KB |
| `brain.glb` | 3D brain model (Allen Brain Atlas, 283 meshes) | ~12MB |

That's it. The entire app is one HTML file plus one 3D model.

---

## Tech Stack

- **Three.js r160** (loaded via CDN importmap) — 3D rendering
- **OrbitControls** — camera rotation/zoom
- **GLTFLoader** — loads the brain.glb model
- **EffectComposer** with UnrealBloomPass + GammaCorrectionShader — glow/bloom post-processing
- **Vanilla JavaScript and CSS** — no framework, no build step

---

## The Four Brain Networks

Each network highlights specific brain meshes in the 3D model. The `networks` object in the script defines everything:

| Network Key | Display Name | Color | Target | Mesh Names |
|-------------|-------------|-------|--------|------------|
| `left-cen` | Founder mode: Directed drive | `#A4C8FF` | Left DLPFC | `Allen_middle_frontal_gyrus_L`, `Allen_superior_frontal_gyrus_L`, `Allen_angular_gyrus_L`, `Allen_supramarginal_gyrus_L` |
| `right-cen` | Deep work: Dial in attention | `#8AB3F9` | Right DLPFC | `Allen_middle_frontal_gyrus_R`, `Allen_superior_frontal_gyrus_R`, `Allen_angular_gyrus_R`, `Allen_supramarginal_gyrus_R` |
| `salience` | Present mind: Less chatter, more clarity | `#FFB974` | dmPFC | `Allen_superior_frontal_gyrus_L/R`, `Allen_paracingulate_gyrus_L/R`, `Allen_long_insular_gyri_L/R` |
| `dmn` | Calm steadiness: Respond over reacting | `#C8D4C8` | dmPFC | `Allen_superior_frontal_gyrus_L/R`, `Allen_precuneus_L/R`, `Allen_cingulate_gyrus_caudal_posterior_part_L/R` |

### Mesh Naming Convention

The brain model uses the Allen Brain Atlas naming: `Allen_[structure]_[L/R]`

- `_L` = brain's anatomical left hemisphere
- `_R` = brain's anatomical right hemisphere

**Important:** The model displays in neurological convention (brain's left appears on the viewer's left when viewed from the front). The `_L` suffix meshes correspond to the Left DLPFC and `_R` meshes to the Right DLPFC. Do not swap these.

---

## Critical Rendering Requirements

These were hard-won through debugging — skipping any of them will cause visual issues:

### 1. Color Management Must Be Disabled
```js
THREE.ColorManagement.enabled = false;
renderer.outputColorSpace = THREE.LinearSRGBColorSpace;
renderer.toneMapping = THREE.NoToneMapping;
```
Without this, network colors shift and don't match the design.

### 2. Post-Processing Chain (Exact Order)
```
RenderPass → UnrealBloomPass → GammaCorrectionShader
```
Do NOT use `OutputPass` instead of `GammaCorrectionShader` — it causes background color mismatch between the canvas and surrounding page.

### 3. Bloom Parameters
```js
new UnrealBloomPass(resolution, 0.18, 0.12, 0.5)
// strength: 0.18, radius: 0.12, threshold: 0.5
```

### 4. Dark Background Required
The scene background is `0x030403` and the page background is `#202621`. The bloom effect requires a dark container to work properly.

---

## Visual Parameters (Full Reference)

All values below are tuned and should be treated as design specs:

### Scene & Renderer
| Parameter | Value |
|-----------|-------|
| Scene background | `0x030403` |
| Page background (CSS) | `#202621` |
| Camera FOV | 50 |
| Camera distance | 1.8 |
| Pixel ratio cap | `Math.min(devicePixelRatio, 2)` |

### Bloom (UnrealBloomPass)
| Parameter | Value |
|-----------|-------|
| Strength | 0.18 |
| Radius | 0.12 |
| Threshold | 0.5 |

### Mesh Opacity
| State | Solid Opacity | Wireframe Opacity |
|-------|--------------|-------------------|
| Base (non-highlighted) | 0.05 | 0.05 |
| Highlighted | 0.9 | 0.7 |

### Emissive / Glow
| Parameter | Value |
|-----------|-------|
| Base emissive intensity | 0.35 |
| Pulse amount | ±0.015 (sine wave) |

### Controls (OrbitControls)
| Parameter | Value |
|-----------|-------|
| Auto-rotate speed | 0.4 |
| Damping factor | 0.08 |
| Min zoom distance | 1.2 |
| Max zoom distance | 3.0 |
| Pan | Disabled |

### Lighting
| Light | Type | Color | Intensity | Position |
|-------|------|-------|-----------|----------|
| Ambient | AmbientLight | `0x404550` | 0.5 | — |
| Key | DirectionalLight | `0xffffff` | 0.6 | (5, 5, 5) |
| Fill | DirectionalLight | `0x88aacc` | 0.4 | (-5, 3, -5) |

### Base Mesh Materials
| Property | Value |
|----------|-------|
| Color | `0x4a5055` |
| Wireframe color | `0x6a7580` |
| Side | DoubleSide |
| depthWrite (base) | false |
| depthWrite (highlighted) | true |

---

## Visual Behavior

- **Default state:** All 283 meshes at 0.05 opacity with 0.05 wireframe opacity (near-invisible ghost brain)
- **Highlighted state:** Selected network meshes at 0.9 opacity with emissive glow (0.35 intensity) + subtle pulse animation
- **Non-highlighted meshes** stay translucent so deeper brain structures (like the insula or cingulate) glow through when selected
- **Auto-rotation** at 0.4 speed, pauses when user drags
- **Zoom** enabled between 1.2x and 3x distance

---

## Educational Videos

Each network links to a curated YouTube clip:

| Network | Expert | YouTube ID | Start Time (s) |
|---------|--------|-----------|-----------------|
| left-cen | Dr. Nolan Williams | `X4QE6t-MkYE` | 2749 |
| right-cen | Dr. Judson Brewer | `1M-bUw-OiG0` | 1596 |
| salience | Dr. Nolan Williams | `X4QE6t-MkYE` | 2916 |
| dmn | Dr. Andrew Huberman | `wTBSGgbIvsY` | 1380 |

Embed URL pattern:
```
https://www.youtube.com/embed/{videoId}?autoplay=1&start={startTime}&rel=0&playsinline=1
```

---

## Integration into Next.js

### Serving the Brain Model

Copy `brain.glb` into your `public/` directory and update the loader path:
```js
// Current (prototype):
loader.load('https://drjeeshan.github.io/pharia-brain/brain.glb', ...)

// Production:
loader.load('/brain.glb', ...)
```

The model is ~12MB — show a loading state while it downloads. The prototype uses a spinner overlay that fades out on load.

### Building a Reusable Component

Since the brain will appear across multiple pages, extract it as a React component:

```tsx
// Suggested props interface
interface BrainVisualizationProps {
  initialNetwork?: 'left-cen' | 'right-cen' | 'salience' | 'dmn';
  showCarousel?: boolean;
  showVideoModal?: boolean;
  height?: string; // e.g. "45vh", "400px"
}
```

This lets you use the same component in different contexts:
- Full interactive version with carousel on the Science page
- Simplified version (single highlighted network, no carousel) on other pages
- Different initial network depending on page context

### Key Extraction Steps

1. **Three.js scene setup** — everything from `THREE.ColorManagement.enabled = false` through the EffectComposer setup goes into a `useEffect` with cleanup
2. **The `networks` object** — extract as a shared constant/config file
3. **`highlightNetwork()` function** — the core logic that sets mesh materials based on selected network
4. **Carousel and video modal** — these are standard React UI, just need the network data
5. **Animation loop** — use `requestAnimationFrame` with proper cleanup on unmount
6. **Resize handling** — attach to window resize, clean up on unmount

### Things to Watch For

- **Dispose Three.js resources on unmount** — renderer, geometries, materials, textures. The prototype doesn't do this since it's a single page, but in a React SPA you'll get memory leaks without cleanup
- **Canvas touch-action** — set to `none` to prevent scroll interference on mobile
- **Pixel ratio** — cap at 2x for performance: `Math.min(window.devicePixelRatio, 2)`
- **Frame rate** — the prototype throttles to ~60fps with a 16ms check in the animation loop

---

## Brand Fonts

The prototype references two custom fonts (loaded from local files not in the repo):

- **Founders Grotesk** (headlines) — weight 300, 400
- **Matter** (body text) — weight 400, 500

The 3D brain itself doesn't depend on fonts — only the carousel cards and UI text use them.

---

## Questions?

The repo is at https://github.com/drjeeshan/pharia-brain. The `v7-carousel.html` file is self-contained — you can open it locally, search for any function name, and see exactly how it works.
