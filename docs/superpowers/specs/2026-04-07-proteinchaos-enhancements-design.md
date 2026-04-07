# ProteinCHAOS Enhancements — Design Spec
**Date:** 2026-04-07  
**Scope:** Single-file (`index.html`) — no build system, no npm, no new files  
**Audience:** Structural biology researchers wanting publication-quality art from protein structures

---

## 1. Physics Performance

### 1.1 Cell-list WCA repulsion (replaces O(N²))
**Current:** Loop over all atom pairs, skipped entirely above 1500 atoms.  
**New:** Build a grid of cells (cell size = WCA cutoff ≈ 1.26 Å in sim units). For each atom, only check atoms in the 27 neighboring cells. O(N) average.  
- Remove the `if (nAtoms > 1500) return;` guard — repulsion works for all system sizes.  
- Cell grid rebuilt once per `init`, stored as `wcaCells: Map<string, Int32Array>`.  
- No inter-frame state needed beyond the cell map.

### 1.2 Precompute rotation matrix once per frame
**Current:** `accumulateChaos()` recomputes 9 trig values per bond × 2 endpoints.  
**New:** Compute the 3×3 rotation matrix (R = Rz·Ry·Rx) once at the top of `accumulateChaos()`, apply via 9 multiplies per endpoint instead of 15 trig calls per bond.  
Matrix elements: `m00…m22` as local variables.

### 1.3 Pre-baked Gaussian noise cache
**Current:** `randNormal()` calls `Math.random()` twice + `Math.sqrt` + `Math.cos` per invocation, called `nAtoms × substeps` times per frame.  
**New:** Allocate `noiseCache = new Float32Array(65536)` on init, fill with Box-Muller once. Use a cursor that wraps. Refill async in a `setTimeout` when cursor nears end.

### 1.4 Ping-pong transferable buffers
**Current:** `new Float32Array(accumulationBuffer)` copies the full buffer every frame (up to 33 MB for 4K RGB).  
**New:** Maintain `bufA` and `bufB` in the worker. When sending, transfer ownership of the active buffer and swap to the other. Main thread transfers it back after `renderDisplay`. Zero allocation per frame after init.  
Protocol addition: main thread sends `{type:'buf_return', buffer: <ArrayBuffer>}` after each render.

---

## 2. Visual Effects & Rendering

### 2.1 ACES filmic tone mapping
Replace `Math.log(val+1)/logDen` with the ACES fitted curve:
```
function aces(x) {
  const a=2.51, b=0.03, c=2.43, d=0.59, e=0.14;
  return clamp((x*(a*x+b))/(x*(c*x+d)+e), 0, 1);
}
```
Apply after normalizing to `[0,1]` by `val/displayMax`. Keep auto-exposure (`displayMax` EMA). Gamma applied after ACES.

### 2.2 Enhanced depth cueing
**Current:** `depthCue = max(0.1, 1 + z * 0.05)` — brightness only.  
**New:** Two effects combined:
- **Intensity falloff:** `brightness = max(0.1, 1 + z * 0.08)` (slightly stronger)
- **Hue shift:** accumulate into a 4-channel buffer (RGBZ). In `renderDisplay`, the Z channel drives a lerp between the atom's color and a "fog color" (theme-specific: cool blue for dark themes, warm cream for light themes). Near atoms are vivid; far atoms fade into the background.  
- Actually simpler: pass a `depthColor` weight into the accumulation: `val *= brightness; hueShift = clamp(1 - brightness, 0, 0.4)` and blend in renderDisplay.

### 2.3 2D canvas bloom pass
After `artCtx.putImageData(viewImageData, 0, 0)`, draw a second pass:
```js
// on an offscreen canvas at 1/4 resolution:
offCtx.filter = 'blur(8px)';
offCtx.drawImage(artCanvas, 0, 0, artWidth/4, artHeight/4);
// composite back:
artCtx.globalCompositeOperation = 'screen';
artCtx.globalAlpha = bloomStrength; // slider 0.0–0.4
artCtx.drawImage(offCanvas, 0, 0, artWidth, artHeight);
artCtx.globalCompositeOperation = 'source-over';
artCtx.globalAlpha = 1.0;
```
Add a "Bloom" slider (0–0.5, default 0.15) to the Style section.

### 2.4 Auto-rotation / slow tumble
Add a `spinSpeed` param (degrees/second, 0 = off). In the main `tick()`:
```js
if (spinSpeed > 0) {
  currentRotY += spinSpeed * dt / 1000;
  ui.rotY.value = currentRotY % 360;
  window.updateWorkerParams();
}
```
Add a "Spin" toggle + speed slider (0–30 °/s, default 5) to the Camera section. Auto-clears the buffer every N degrees to let trails accumulate cleanly.

### 2.5 Proper film grain (value noise)
Replace sin-hash with a 256×256 tileable value-noise map:
- Generate on init: smooth bilinear-interpolated random grid at 32×32 resolution, upsample to 256×256.
- Tile with `(x % 256) + (y % 256) * 256`.
- Brightness-adaptive: `grainStrength = baseGrain * (1 - pixel_brightness * 0.7)` — bright areas nearly grain-free, dark areas get full grain. Matches film behavior.

---

## 3. Chain Coloring & Color System

### 3.1 Per-chain color pickers
On structure load, detect unique chains and render a row per chain in a new "Chains" section of the sidebar:
```html
<div class="chain-row">
  <span class="chain-label">Chain A</span>
  <input type="color" value="#93c5fd" oninput="updateChainColor('A', this.value)">
  <span class="chain-atom-count">142 res</span>
</div>
```
`updateChainColor(chainId, hexColor)`:
1. Update `window.chainColors[chainId]` (hex → THREE.Color → rgb float).
2. Rebuild `colorArray` from `window.globalAtoms` using `window.chainColors`.
3. Post `{type:'colors', data: colorArray}` to worker.
4. Call `update3DColors()`.

Chain color state lives in `window.chainColors = {}` (chain letter → hex string).

### 3.2 New curated palettes
Add to `CHAIN_PALETTES`:
- `'vivid'`: High-saturation 8-color (for cover art, dark backgrounds)
- `'publication'`: Grays + accents suitable for B&W print
- `'wong'`: 8-color colorblind-safe set (Wong 2011)
- `'pastel'`: Soft 8-color for presentations

### 3.3 Residue-property coloring
New color mode `colorMode = 'residue'`. Color mapping:
- Hydrophobic (ALA, VAL, ILE, LEU, MET, PHE, TRP, PRO): `#f97316` (orange)
- Polar uncharged (SER, THR, CYS, TYR, ASN, GLN): `#22d3ee` (cyan)
- Positive (LYS, ARG, HIS): `#3b82f6` (blue)
- Negative (ASP, GLU): `#ef4444` (red)
- Glycan: `#f59e0b` (amber)
- Nucleic: `#a78bfa` (purple)

Separate from chain-color mode — selectable via a new dropdown in the Color section.

---

## 4. UX Fixes

### 4.1 Chain-color mode decoupled from theme
Remove the `theme.startsWith('chain')` check. Instead, add a `colorMode` dropdown ("Density" / "Chain" / "Residue") in the Color section. Selecting "Chain" or "Residue" sets `colorMode` independently of the visual theme.

### 4.2 BAOAB integrator fix
The second half-kick in `integrate()` uses stale forces (from before the noise/friction step). Fix: move `calcForces()` call to after the second position half-step, before the second velocity half-kick. This makes it a proper BAOAB scheme.

### 4.3 Adaptive substep cap
Raise from `Math.min(20, ...)` to `Math.min(40, ...)`. Also add a "Max substeps" display in the physics section.

### 4.4 Clipboard copy button
Add a "Copy" button next to "Save" that calls `artCanvas.toBlob(blob => navigator.clipboard.write([new ClipboardItem({'image/png': blob})]))`.

---

## 5. Architecture Notes

All changes stay within `index.html`. The worker code string `workerCode` gets:
- Cell-list WCA
- Rotation matrix precompute
- Noise cache
- Ping-pong buffer protocol
- ACES tone mapping inputs (no change needed — ACES applied in main thread)
- Residue-property color mode

Main thread gets:
- ACES tone mapping in `renderDisplay()`
- Bloom pass after `putImageData`
- Auto-spin in `tick()`
- Film grain replacement in `setupCanvasResolution()`
- Per-chain color picker UI + `updateChainColor()`
- Palette additions
- UX fixes

No new external dependencies. Bloom uses the existing `artCanvas` + a new small offscreen canvas.

---

## 6. What This Doesn't Change

- Single-file architecture
- Three.js CDN version (0.160)
- Tailwind CSS CDN
- PDB parsing logic
- Glycan topology (LINK records + anchor pass)
- Export formats (PNG, WebM/MP4)
- Boomerang recording mode
