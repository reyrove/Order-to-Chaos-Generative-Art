# Order-to-Chaos-Generative-Art

A seed-based generative art catalogue for **Order to Chaos**, a computational gradient-grid system. Built as a single-file HTML page with p5.js, styled as a gallery-ready portfolio and licensing surface.

**Live:** `https://reyrove.github.io/Order-to-Chaos-Generative-Art/`

---

## Overview

**Order to Chaos** renders an 80 × 80 grid from numeric seeds. Cells drift horizontally from a strict sine/cosine lattice on the left to noise-driven disorder on the right. Every lit cell draws itself as a rectangle, ellipse, or triangle in a small harmonic palette — complementary, analogous, or triadic — chosen per seed.

The page functions as a **catalogue and licensing surface** for fashion houses, textile studios, and surface designers. Every composition is reproducible by seed, meaning the same number always produces the identical grid.

---

## Features

### Generative Engine (p5.js)

- **80 × 80 grid** — 6,400 cells per composition, each independently lit or dark
- **Order → Chaos gradient** — a horizontal blend factor `t = i / (cols − 1)` interpolates between a strict lattice and Perlin-driven noise
  - **Order side:** `sin(nx·10) · cos(ny·10)` plus a radial field modulated by `sin(distance · π · 2)`
  - **Chaos side:** 2D value-noise at 5× frequency plus a flat random chaos term up to 0.6
- **Three cell shapes** — rectangle, ellipse, or triangle at random size (0.4–1.6× cell) and rotation (up to 90°)
- **Per-seed harmonic palettes** — a base hue plus three companions in one of:
  - **Complementary** (base, +180°, +360°, +540°)
  - **Analogous** (base, +30°, +60°, +90°)
  - **Triadic** (base, +120°, +240°, +360°)
- **Per-cell modulation** — hue varies ±20°, saturation ±10, brightness ±10
- **Scattered glow** — cells in even rows/columns spawn soft ellipses at 10% alpha, accumulating into a diffuse wash
- **Dim background wash** — a single hue at low saturation (15–30) and low brightness (10–20), overlaid at 15% alpha

### Same Seed → Many Surfaces

A single seed drives every instance on the page:

| Instance | Purpose | Tiling |
|----------|---------|--------|
| **Cover** | Hero composition | 1 × 1 |
| **Plate** | Main work (master) | 1 × 1 |
| **Frame** | Framed archival print | 1 × 1 |
| **Print** | Archival print surface | 1 × 1 |
| **Scarf** | Silk scarf (3:1) | 3 × 1 |
| **Textile** | Fabric yardage (4:3) | 2 × 2 |
| **Wall** | Wallpaper (2:3) | 1 × 2 |
| **Archive** | 8 unique seeds | 1 × 1 each |

### Controls

- **New Seed** button / `R` key — regenerate with a fresh seed
- **Download** button / `S` key — export the plate canvas as PNG
- **Click any archive plate** — load that seed into the main view

### Performance

- Debounced resize handling (250ms)
- `IntersectionObserver` pauses off-screen sketches
- `visibilitychange` pauses rendering when the tab is hidden
- `pixelDensity(1)` for a lighter canvas footprint
- Two-frame render (`maxFrames: 2`) so tiles draw once and stop

---

## File Structure

```
Order-to-Chaos-Generative-Art/
├── index.html          # Single-file page (structure + styles + generative engine)
├── README.md
└── images/
    ├── fav.svg         # Favicon
    ├── tote.png        # Mockup: tote bag
    ├── tee.png         # Mockup: t-shirt
    └── cushion.png     # Mockup: cushion
```

No build step. No dependencies to install. p5.js is loaded from a CDN.

---

## Getting Started

1. Clone the repo:
   ```bash
   git clone https://github.com/reyrove/Order-to-Chaos-Generative-Art.git
   cd Order-to-Chaos-Generative-Art
   ```
2. Add the three mockup images to `images/` (optional — they appear in the Commission section).
3. Open `index.html` in a browser.

That's it. The page self-initialises on `DOMContentLoaded`.

---

## How the Generative System Works

### Seeded Random

A linear congruential generator replaces p5's native randomness inside each sketch:

```js
class SeededRandom {
  constructor(s) { this.state = (Math.abs(s) % 2147483647) || 1; }
  rand() { this.state = (this.state * 9301 + 49297) % 233280; return this.state / 233280; }
  range(min, max) { return this.rand() * (max - min) + min; }
  int(min, max) { return Math.floor(this.range(min, max + 1)); }
  pick(arr) { return arr[Math.floor(this.rand() * arr.length)]; }
}
```

Every aesthetic decision — background hue, palette harmony, grid state, cell shape, cell colour — is drawn from this generator in a fixed order. Same seed, same sequence, same composition.

### Parameter Space (per seed)

| Parameter | Range |
|-----------|-------|
| Background hue | 0 – 360 |
| Background saturation | 15 – 30 |
| Background brightness | 10 – 20 |
| Base hue | 0 – 360 |
| Palette harmony | Complementary / Analogous / Triadic |
| Palette companion hues | 4 total (base + 3) |
| Palette saturation | 60 – 90 |
| Palette brightness | 60 – 90 |
| Shape type | Rect / Ellipse / Triangle |
| Shape size | 0.4 – 1.6× cell |
| Shape rotation | 0 – 90° |

### The Order → Chaos Gradient

Each cell's lit/dark state blends two fields along a horizontal axis:

```js
const t = i / (COLS - 1);   // 0 at left, 1 at right

// ORDER component
const orderVal = Math.sin(nx * 10) * Math.cos(ny * 10) * 0.5 + 0.5;
const orderThreshold = 0.45 + Math.sin(distance * Math.PI * 2) * 0.2;

// DISORDER component
const disorderVal = sampleNoise(noiseTable, nx * 5, ny * 5);
const disorderThreshold = 0.5 + (sr.rand() * 0.6) - 0.3;

// Blend
const blendedVal       = orderVal       * (1 - t) + disorderVal       * t;
const blendedThreshold = orderThreshold * (1 - t) + disorderThreshold * t;

grid[i][j] = blendedVal > blendedThreshold ? 1 : 0;
```

The left edge reads as a woven lattice; the right edge reads as noise; the middle is a smooth drift between the two. No two rows behave identically.

### Seeded Value-Noise

The original sketch uses p5's `noise()`, which is seeded globally and not affected by `setSeeds()`. To make each seed produce a different composition, this adaptation replaces it with a small seeded value-noise function:

```js
function buildNoiseTable(sr) {
  const size = 256;
  const table = new Float32Array(size);
  for (let i = 0; i < size; i++) table[i] = sr.rand();
  return table;
}

function sampleNoise(table, x, y) {
  // Bilinear interpolation with smoothstep fade
  // ...
}
```

Same visual character, fully deterministic per seed.

### Tiling Logic

Rectangular surfaces (scarf, textile, wall) tile the square composition N×M. Each tile is drawn into its own translated coordinate space, centred within its cell:

```js
const tileW = canvasW / tileCols;
const tileH = canvasH / tileRows;
const tileSize = Math.min(tileW, tileH);
// draw square of side tileSize at (ox, oy) for each cell
```

This preserves the square composition's proportions regardless of surface aspect ratio.

---

## Customisation

### Change the default seed

Edit the init function:

```js
function init() {
  seed = 123456789;  // fixed seed instead of random
  // ...
}
```

### Adjust the grid resolution

Edit the constants near the top of the script:

```js
const COLS = 80;
const ROWS = 80;
```

Higher values produce finer mosaics. Both dimensions must divide evenly into the canvas for clean rendering.

### Adjust the order/chaos balance

Find the grid initialisation inside `drawComposition`. Key knobs:

- **`t = i / (COLS - 1)`** — the blend axis. Change to `j / (ROWS - 1)` for a vertical gradient, or use `(i + j) / (COLS + ROWS - 2)` for a diagonal.
- **`Math.sin(nx * 10) * Math.cos(ny * 10)`** — the order-side lattice frequency. Change `10` to raise or lower the weave density.
- **`sampleNoise(noiseTable, nx * 5, ny * 5)`** — the chaos-side noise frequency. Change `5` to zoom the noise field in or out.
- **`sr.rand() * 0.6`** — the chaos amplitude. Higher values push the right edge further into disorder.

### Add a fourth palette harmony

Edit the harmony switch inside `drawComposition`:

```js
if (harmony === 0) hue = (baseHue + 180 * i) % 360;      // complementary
else if (harmony === 1) hue = (baseHue + i * 30) % 360;  // analogous
else hue = (baseHue + i * 120) % 360;                    // triadic
```

Add a branch and update the `sr.int(0, 2)` call to `sr.int(0, 3)`. Also extend `HARMONY_NAMES` so the metadata display picks it up.

### Change surface tiling

Edit `surfaceDefs`:

```js
const surfaceDefs = [
  { id: 'surfacePrint',   wrapId: 'wrap-surfacePrint',   tile: { cols: 1, rows: 1 } },
  { id: 'surfaceScarf',   wrapId: 'wrap-surfaceScarf',   tile: { cols: 3, rows: 1 } },
  // ...
];
```

The `tile` object accepts any `{ cols, rows }` pair.

### Replace mockup images

Swap the files in `images/` or update the `src` attributes in the Commission section:

```html
<img src="images/your-mockup.png" alt="..." loading="lazy" />
```

---

## Design System

| Token | Value | Use |
|-------|-------|-----|
| `--bg` | `#0a0a0a` | Page background |
| `--paper` | `#f4f1ea` | Light surface (Surfaces section) |
| `--gold` | `#b8935a` | Primary accent |
| `--gold-soft` | `#d4b483` | Italic emphasis |
| `--serif` | Cormorant Garamond | Headings, titles |
| `--mono` | DM Mono | Labels, metadata, UI |

Type scale, spacing, and section rhythm match the Skyline, Portrait, Girih, and Gothic Grid templates. Responsive breakpoints at 900px, 720px, 560px, and 400px.

---

## Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `R` | Regenerate with a new seed |
| `S` | Download plate as PNG |
| `Enter` / `Space` | Activate focused archive item |
| `Esc` | Close legal modal |

---

## Browser Support

Modern evergreen browsers with:

- Canvas 2D (universal)
- `IntersectionObserver` (universal in modern browsers)
- `backdrop-filter` (Safari 9+, Chrome 76+, Firefox 103+)
- CSS custom properties (universal)

Fallback: if `IntersectionObserver` is unavailable, sketches render continuously.

---

## Accessibility

- Archive items are keyboard-focusable with `role="button"` and `aria-label`
- Legal modal traps focus, closes on `Esc` and overlay click
- `prefers-reduced-motion` disables transitions and smooth scroll
- All interactive elements have visible focus states (via browser default)

---

## Section Map

| Section | ID | Purpose |
|---------|-----|---------|
| Cover | — | Hero title + live grid |
| Statement | `#statement` | Artist statement |
| Plate | `#work` | Main work + metadata + controls |
| Surfaces | `#application` | Framed print + 4 tiled surfaces |
| Process | `#process` | Three process cards |
| Archive | `#archive` | 8 selectable seeds |
| Commission | `#commission` | Services + mockups + CTA |
| Colophon | — | Footer, links, legal |
| Legal modal | `#legalModal` | Licensing / Terms / Credits |

---

## Deployment (GitHub Pages)

1. Push the repo to GitHub.
2. Go to **Settings → Pages**.
3. Under **Source**, select `Deploy from a branch`.
4. Choose branch `main` and folder `/ (root)`.
5. Save — the site goes live at `https://reyrove.github.io/Order-to-Chaos-Generative-Art/`.

---

## Notes on the Source Sketch

The generative engine is adapted almost verbatim from the original p5 sketch. The adaptations that turn it into a catalogue page:

1. **Seeded random replaces p5's `random()`** — a local `SeededRandom` class drives every choice. The original sketch calls `setSeeds()` for fxhash compatibility; that's replaced with the seeded generator directly.
2. **Sketch factory** — `makeSketch({ slotId, seedValue, ... })` returns a p5 sketch that can be instantiated into any DOM slot, enabling cover / plate / frame / surfaces / archive from one seed.
3. **DOM-driven canvas sizing** — `cs = min(windowHeight, windowWidth)` becomes slot-rect dimensions.
4. **Tiling support** — the `tile: { cols, rows }` parameter repeats the square composition across rectangular surfaces.
5. **Drawing logic takes `p` explicitly** — the original used p5's global mode; here `p` is passed so the factory closure works.
6. **Seeded value-noise replaces p5's `noise()`** — p5's Perlin noise is seeded globally and not affected by `setSeeds()`. A small value-noise function built from the same `SeededRandom` table restores per-seed reproducibility without changing the visual character.
7. **`constrain` calls inlined** — replaced with `Math.max`/`Math.min` so the drawing code doesn't depend on p5 globals inside the factory.
8. **`windowResized` replaced by a debounced `handleResize`** — all sketches are destroyed and recreated on resize (250ms debounce), since their canvases are locked to slot dimensions.
9. **`triggerPreview` removed** — the original calls it for fxhash thumbnail generation; not needed in the plain-browser catalogue.

---

## Related Repositories

| Repo | Theme |
|------|-------|
| [`Skyline-Series`](https://github.com/reyrove/Skyline-Series) | Themed cityscapes |
| [`Portrait-Series-Generative-Art`](https://github.com/reyrove/Portrait-Series-Generative-Art) | Generative faces |
| [`Girih-2-Generative-Art`](https://github.com/reyrove/Girih-2-Generative-Art) | Islamic geometric ornament |
| [`Gothic-Grid-Generative-Art`](https://github.com/reyrove/Gothic-Grid-Generative-Art) | Mosaic grid composition |
| **`Order-to-Chaos-Generative-Art`** | Gradient grid composition *(this repo)* |

All five share the same catalogue template, design tokens, and seed-based reproducibility model.

---

## Credits

- **Design & Generative System** — Reyhaneh Daneshdoost
- **Typefaces** — Cormorant Garamond, DM Mono
- **Engine** — p5.js 1.9.0
- **Edition** — Order to Chaos, Autumn 2026

---

## License

All artwork, code, and generative systems are the intellectual property of the artist. Compositions may not be reproduced, redistributed, resold, or adapted without a written license.

For licensing, commissions, or private systems: **reyhanehdaneshdoost@gmail.com**