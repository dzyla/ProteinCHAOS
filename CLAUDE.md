# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ProteinCHAOS is a single-page browser app that loads protein structures (PDB/AlphaFold), runs a coarse-grained stochastic dynamics simulation in a Web Worker, and renders long-exposure "chaos art" trails via a 2D accumulation buffer. It is entirely self-contained in `index.html` — no build system, no npm, no bundler.

## Development

Since there is no build step, development is just editing `index.html` and opening it in a browser. To serve locally:

```bash
python3 -m http.server 8080
# then open http://localhost:8080
```

Deployment is automatic: pushing to `main` triggers the GitHub Actions workflow (`.github/workflows/static.yml`) that deploys to GitHub Pages.

## Architecture (all in `index.html`)

The file has three logical sections inside a single `<script type="module">`:

### 1. Worker Code (lines ~721–1093, embedded as a JS string `workerCode`)
The physics engine runs entirely inside a Web Worker created from a Blob URL. It contains:
- **`CONF`** — physics constants (friction, timestep, bond stiffness, WCA repulsion)
- **`integrate()`** — Langevin dynamics stepper (Euler with friction + thermal noise)
- **`calcForces()`** — harmonic bonds, WCA excluded volume, positional restraints for protein/DNA/glycan
- **`accumulateChaos()`** — projects 3D atom positions onto a 2D RGBA accumulation buffer (the "long exposure" effect)
- **`fadeAccumulationBuffer()`** — block-wise buffer decay between frames
- **`onmessage`** — message router: `init`, `start`, `stop`, `updateParams`, `reset`, `resize`, `setColorMode`

The worker communicates back to the main thread by transferring a SharedArrayBuffer/typed array of accumulated pixel data.

### 2. Main Thread — Constants & Data (lines ~1196–1300)
- **`PRESETS`** — named color themes (fire, ice, graphite, etc.), each defining palette + background
- **`RESIDUE_MASSES`** — coarse-grained mass table per residue type
- **`GLYCAN_NAMES`** — set of recognized glycan residue names
- **`CHAIN_PALETTES`** — per-chain color arrays for Three.js preview

### 3. Main Thread — Application Logic (lines ~1300+)
Key functions:
- **`init()`** — creates the Worker from `workerCode` Blob, sets up message handler
- **`tick()`** — `requestAnimationFrame` loop; sends `step` messages to worker, calls `renderDisplay`
- **`buildRestraintsSpatial()`** — parses loaded atoms, builds bond topology and spatial restraints using a grid hash for neighbor search
- **`handleWorkerMessage()`** — receives accumulated buffer from worker, triggers render + video recording
- **`renderDisplay()`** — draws accumulation buffer onto `#art-canvas`, applies watermarks
- **`applyTheme()` / `getThemeBackground()`** — maps preset name → colors + CSS background
- **`setupThreeJS()`** — Three.js scene with ArcballControls, SSAO, UnrealBloom for the 3D structure preview panel
- **`bindUI()`** — wires all sidebar sliders/buttons/toggles to worker `updateParams` messages

### UI Layout
- **`#sidebar`** (360px fixed) — all controls: structure input, physics sliders, style presets, export
- **`#canvas-3d`** (top 40% of main area) — Three.js 3D preview with orbit controls
- **`#art-panel`** (bottom 60%) — 2D chaos art canvas

## Key Constraints
- **No external JS dependencies except CDN**: Three.js r0.160 and Tailwind CSS v4 are loaded from jsDelivr at runtime. Do not introduce npm packages or local bundling.
- **Worker code is a string**: The worker runs from `URL.createObjectURL(new Blob([workerCode]))`. Any worker changes must be made inside the `workerCode` template literal, not as a separate file.
- **SharedArrayBuffer / transferable**: The accumulation buffer is passed back via `postMessage` with a transferable. Be careful with ownership when modifying the message protocol.
- **Single-file constraint**: All logic, styles, and markup live in `index.html`. Keep it that way unless there is a strong reason to split.
