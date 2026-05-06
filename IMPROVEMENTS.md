# IMPROVEMENTS.md — ProteinCHAOS v1.3 Implementation Plan

> **Purpose:** Actionable task list for an AI coding worker to implement improvements to ProteinCHAOS.
> Each task includes rationale, step-by-step implementation instructions with code-level references,
> acceptance criteria, complexity estimate, and dependency graph.
>
> **Priority Legend:** 🟢 Quick win | 🟡 Moderate effort | 🔴 Major effort
>
> **Complexity Scale:** S (< 1 hour) | M (1–4 hours) | L (4–12 hours) | XL (> 12 hours)

---

## Codebase Reference

ProteinCHAOS is a **single HTML file** (`index_improved(2).html`, ~2700 lines) containing:

| Section | Location | Description |
|---------|----------|-------------|
| CSS styles | `<style>` block in `<head>` | Dark glass theme, layout, controls, mobile |
| HTML sidebar | `<div id="sidebar">` | All UI controls, sliders, buttons |
| HTML viewports | `<div id="main-view">` | 3D preview (`#canvas-3d`) + art panel (`#art-panel`) |
| Worker code | `const workerCode = \`...\`` | String literal containing entire Web Worker JS |
| Main app JS | `<script type="module">` after worker | Three.js setup, parsers, rendering, UI bindings |

**Key variables/functions in Worker:**
- `CONF` — physics constants object
- `integrate()` — Langevin dynamics step
- `calcForces()` — all force terms (bonds, restraints, WCA, glycan repulsion)
- `accumulateChaos()` — point scattering into accumulation buffer
- `fadeAccumulationBuffer()` — exponential decay
- `buildWCACells()` — spatial hashing for WCA
- `posX/Y/Z`, `velX/Y/Z`, `forceX/Y/Z` — flat Float32Arrays (SoA layout)
- `bonds[]`, `restraints[]` — topology arrays
- `flexibility[]` — per-atom flexibility factors
- `isNucleic[]`, `isGlycan[]` — per-atom type flags

**Key variables/functions in Main Thread:**
- `parseAndLoad(txt)` — master parser, topology builder, worker init
- `parseCIF(txt)` — mmCIF parser
- `buildRestraintsSpatial(atoms)` — elastic network construction
- `renderDisplay(buffer)` — tone mapping + theme + bloom + grain pipeline
- `accesToneMap(x)` — ACES filmic curve
- `applyTheme(theme, t, bg)` — color ramp functions
- `handleWorkerMessage(e)` — processes worker→main messages
- `window.globalAtoms`, `window.globalBonds` — parsed structure data
- `worker` — Web Worker instance
- `artCanvas`, `artCtx`, `artWidth`, `artHeight` — 2D art canvas
- `RESIDUE_MASSES`, `GLYCAN_NAMES`, `RESIDUE_PROPERTIES` — lookup tables

**Constraints:**
- All code must remain in a single HTML file (no build step)
- Worker code is a string literal — edits require escaping backticks and `${}`
- External libraries loaded via CDN ESM imports
- No network access from worker (blob URL worker)

---

## Phase 1 — Quick Wins 🟢

*Low-risk, high-value improvements that can be implemented independently.*

---

### Task 1.1: Parse SSBOND Records for Disulfide Restraints

**Priority:** 🟢 | **Complexity:** S | **Dependencies:** None

**Rationale:** Disulfide bonds (SSBOND records in PDB) are strong covalent crosslinks that significantly constrain protein dynamics. Currently ignored, causing unnatural flexibility between linked cysteines.

**Implementation:**

1. **In `parseAndLoad()` (PDB parsing loop),** add SSBOND parsing alongside LINK parsing:
```javascript
// Add before the line: if (l.startsWith('ATOM') || l.startsWith('HETATM'))
if (l.startsWith('SSBOND')) {
    // PDB SSBOND format: cols 16(chain1), 18-21(resNum1), 30(chain2), 32-35(resNum2)
    const c1 = l[15], r1 = l.substring(17, 21).trim();
    const c2 = l[29], r2 = l.substring(31, 35).trim();
    linkRecords.push({ c1, r1, c2, r2, type: 'ssbond' });
    continue;
}
```

2. **In the bond creation section,** after LINK record processing, handle SSBOND entries by creating bonds with `visualWeight: 0.8` and a strong stiffness. Modify the LINK loop to also accept non-glycan SSBOND pairs:
```javascript
for (let lnk of linkRecords) {
    const idx1 = atomMap.get(`${lnk.c1}_${lnk.r1}`);
    const idx2 = atomMap.get(`${lnk.c2}_${lnk.r2}`);
    if (idx1 !== undefined && idx2 !== undefined) {
        const dist = Math.hypot(...);
        if (lnk.type === 'ssbond') {
            newBonds.push({ i: idx1, j: idx2, visualWeight: 0.8, d: dist });
            uf.union(idx1, idx2);
        } else if (newAtoms[idx1].isGlycan || newAtoms[idx2].isGlycan) {
            // existing glycan LINK logic
        }
    }
}
```

3. **For mmCIF:** SSBOND entries are already captured by `_struct_conn` with type `disulf`, which is already included in the filter. Verify that the existing `parseCIF` correctly maps these to `linkRecords`. Add the `type` field:
```javascript
linkRecords.push({ c1: ..., r1: ..., c2: ..., r2: ..., type: t });
```

**Acceptance Criteria:**
- Load `1IGT` (antibody with multiple SS bonds) — disulfide-linked cysteines should show strong bonds in 3D preview and restrained motion in simulation
- Non-glycan LINK records of type `ssbond`/`disulf` create bonds with `visualWeight: 0.8`
- No change to glycan bonding behavior

---

### Task 1.2: Shareable Preset URLs

**Priority:** 🟢 | **Complexity:** S | **Dependencies:** None

**Rationale:** Users can't share their visual configurations. Encoding all parameters in the URL enables reproducible art and social sharing.

**Implementation:**

1. **Create `encodeStateToURL()` function** in main thread JS:
```javascript
function encodeStateToURL() {
    const params = new URLSearchParams();
    params.set('pdb', document.getElementById('pdb-id').value);
    params.set('theme', ui.theme.value);
    params.set('temp', ui.temp.value);
    params.set('k', ui.k.value);
    params.set('speed', ui.speed.value);
    params.set('zoom', ui.zoom.value);
    params.set('decay', ui.decay.value);
    params.set('gamma', ui.gamma.value);
    params.set('softness', ui.softness.value);
    params.set('intensity', ui.intensity.value);
    params.set('jitter', ui.jitter.value);
    params.set('bloom', ui.bloom ? ui.bloom.value : '0.15');
    params.set('depthfog', ui.depthfog ? ui.depthfog.value : '0.3');
    params.set('source', ui.source.value);
    params.set('cmode', document.getElementById('art-color-mode')?.value || 'density');
    params.set('rx', ui.rotX.value);
    params.set('ry', ui.rotY.value);
    params.set('rz', ui.rotZ.value);
    return window.location.origin + window.location.pathname + '?' + params.toString();
}
```

2. **Add a "Share" button** in the Data Actions section of sidebar HTML:
```html
<button onclick="copyShareURL()" class="btn btn-secondary text-xs" title="Copy shareable URL">Share</button>
```

3. **Implement `copyShareURL()`:**
```javascript
window.copyShareURL = () => {
    const url = encodeStateToURL();
    navigator.clipboard.writeText(url)
        .then(() => { /* briefly flash button text to "Copied!" */ })
        .catch(() => alert("Could not copy URL"));
};
```

4. **Expand `init()` URL parameter parsing** (currently only handles `pdb`/`af`):
```javascript
// After: if (pdbParam) document.getElementById('pdb-id').value = ...
const p = urlParams;
if (p.has('theme')) ui.theme.value = p.get('theme');
if (p.has('temp'))  { ui.temp.value = p.get('temp'); ui.tempVal.textContent = parseFloat(p.get('temp')).toFixed(2); }
if (p.has('k'))     { ui.k.value = p.get('k'); ui.kVal.textContent = parseFloat(p.get('k')).toFixed(1); }
// ... repeat for all parameters
if (p.has('rx')) ui.rotX.value = p.get('rx');
if (p.has('ry')) ui.rotY.value = p.get('ry');
if (p.has('rz')) ui.rotZ.value = p.get('rz');
updateArtPanelBackground();
```

**Acceptance Criteria:**
- Clicking "Share" copies a URL to clipboard
- Opening that URL in a new tab loads the same structure with identical visual settings
- All slider values, theme, color mode, camera angles are preserved
- Backward compatible: existing `?pdb=` URLs still work

---

### Task 1.3: Accumulation Progress Indicator

**Priority:** 🟢 | **Complexity:** S | **Dependencies:** None

**Rationale:** Users have no feedback on how much accumulation has built up. A subtle frame counter helps them know when to snapshot.

**Implementation:**

1. **Add a counter element** in `#art-panel` HTML, next to the existing `#sim-timer`:
```html
<div id="accum-counter" class="absolute top-4 left-4 font-mono text-gray-200/30 font-bold text-xs pointer-events-none">
    frames: 0
</div>
```

2. **Update in `handleWorkerMessage()`** where `frameCounter++` is already incremented:
```javascript
const accumEl = document.getElementById('accum-counter');
if (accumEl) accumEl.innerText = `frames: ${frameCounter}`;
```

3. **Reset in `clearArt()`:**
```javascript
const accumEl = document.getElementById('accum-counter');
if (accumEl) accumEl.innerText = 'frames: 0';
```

**Acceptance Criteria:**
- Frame counter visible in top-left of art panel, very subtle (30% opacity)
- Increments during simulation, resets on clear
- Does not appear in exported/downloaded images (it's a DOM overlay, not canvas)

---

### Task 1.4: Touch Gestures on Art Canvas

**Priority:** 🟢 | **Complexity:** M | **Dependencies:** None

**Rationale:** The art panel has `touch-action: none` and `cursor: crosshair` but no touch handlers. Mobile users can't pan or zoom the 2D art view.

**Implementation:**

1. **Add touch state variables** in main thread JS:
```javascript
let touchState = { active: false, lastDist: 0, lastMidX: 0, lastMidY: 0 };
```

2. **Bind touch events to `#art-panel`:**
```javascript
const artPanel = document.getElementById('art-panel');

artPanel.addEventListener('touchstart', (e) => {
    if (e.touches.length === 2) {
        e.preventDefault();
        const t = e.touches;
        touchState.active = true;
        touchState.lastDist = Math.hypot(t[1].clientX - t[0].clientX, t[1].clientY - t[0].clientY);
        touchState.lastMidX = (t[0].clientX + t[1].clientX) / 2;
        touchState.lastMidY = (t[0].clientY + t[1].clientY) / 2;
    }
}, { passive: false });

artPanel.addEventListener('touchmove', (e) => {
    if (!touchState.active || e.touches.length !== 2) return;
    e.preventDefault();
    const t = e.touches;
    const dist = Math.hypot(t[1].clientX - t[0].clientX, t[1].clientY - t[0].clientY);
    const midX = (t[0].clientX + t[1].clientX) / 2;
    const midY = (t[0].clientY + t[1].clientY) / 2;

    // Pinch-to-zoom
    const zoomDelta = (dist - touchState.lastDist) * 0.3;
    const newZoom = Math.max(5, Math.min(200, parseFloat(ui.zoom.value) + zoomDelta));
    ui.zoom.value = Math.round(newZoom);
    ui.zoomVal.textContent = Math.round(newZoom);

    // Two-finger pan
    const panDeltaX = midX - touchState.lastMidX;
    const panDeltaY = midY - touchState.lastMidY;
    ui.panX.value = parseFloat(ui.panX.value) + panDeltaX;
    ui.panY.value = parseFloat(ui.panY.value) + panDeltaY;

    touchState.lastDist = dist;
    touchState.lastMidX = midX;
    touchState.lastMidY = midY;

    window.updateWorkerParams();
    window.clearArt(true); // preserve history
}, { passive: false });

artPanel.addEventListener('touchend', () => { touchState.active = false; });
```

3. **Add single-finger rotate** (optional enhancement):
```javascript
// Single-finger horizontal drag → yaw rotation
artPanel.addEventListener('touchmove', (e) => {
    if (e.touches.length !== 1 || touchState.active) return;
    // Track delta from last single-touch position, map to rotY
}, { passive: false });
```

**Acceptance Criteria:**
- Two-finger pinch zoom changes the zoom slider value and updates the art
- Two-finger pan shifts the pan X/Y values
- Art clears and re-accumulates after gesture (zoom/pan change requires re-render)
- No interference with sidebar scrolling on mobile

---

### Task 1.5: B-factor / pLDDT-Based Flexibility

**Priority:** 🟢 | **Complexity:** M | **Dependencies:** None

**Rationale:** The current contact-density flexibility heuristic ignores experimental data already present in PDB files. B-factors encode per-residue thermal mobility; AlphaFold's pLDDT scores encode confidence (inverse of disorder). Using these produces physically accurate flexibility distributions.

**Implementation:**

1. **Extend atom data with B-factor during parsing.**

   In PDB parser (inside `parseAndLoad`), extract B-factor from columns 61-66:
```javascript
const bfactor = parseFloat(l.substring(60, 66)) || 0;
newAtoms.push({ ..., bfactor });
```

   In `parseCIF`, extract from `B_iso_or_equiv` column:
```javascript
const bfactor = parseFloat(row['B_iso_or_equiv'] || '0');
newAtoms.push({ ..., bfactor });
```

2. **Detect AlphaFold structures** (pLDDT is stored in B-factor column, range 0-100):
```javascript
// Heuristic: if all B-factors are in [0, 100] and mean > 40, likely pLDDT
const bfacs = newAtoms.map(a => a.bfactor).filter(b => b > 0);
const meanB = bfacs.reduce((a, b) => a + b, 0) / bfacs.length;
const isAlphaFold = bfacs.every(b => b >= 0 && b <= 100) && meanB > 40;
```

3. **Pass B-factors to worker** in the `init` message:
```javascript
// In atoms array sent to worker:
atoms[i].bfactor = newAtoms[i].bfactor;
atoms[i].isAlphaFold = isAlphaFold;
```

4. **Modify flexibility calculation in worker** (replacing or blending with contact-density):
```javascript
// In the 'init' handler, after current flexibility calculation:
const hasBfactors = atoms.some(a => a.bfactor > 0);
if (hasBfactors) {
    // Normalize B-factors to [0, 1]
    let bMin = Infinity, bMax = -Infinity;
    for (let i = 0; i < nAtoms; i++) {
        const b = atoms[i].bfactor || 0;
        if (b < bMin) bMin = b;
        if (b > bMax) bMax = b;
    }
    const bRange = (bMax - bMin) || 1;

    for (let i = 0; i < nAtoms; i++) {
        const bNorm = ((atoms[i].bfactor || bMin) - bMin) / bRange; // 0=rigid, 1=flexible
        let bFlex;
        if (atoms[i].isAlphaFold) {
            // pLDDT: high value = confident = rigid, low = disordered = flexible
            bFlex = 0.3 + (1.0 - bNorm) * 1.7; // invert: low pLDDT → high flex
        } else {
            // B-factor: high value = more mobile
            bFlex = 0.3 + bNorm * 1.7;
        }
        // Blend with contact-density flex (70% B-factor, 30% contact)
        flexibility[i] = 0.7 * bFlex + 0.3 * flexibility[i];
        if (isGlycan[i]) flexibility[i] = Math.max(flexibility[i], 1.5);
    }
}
```

**Acceptance Criteria:**
- Load a high-resolution X-ray structure (e.g., `1UBQ`) — loops should flex more than core beta-sheet
- Load an AlphaFold model (e.g., `AF-Q5VSL9`) — disordered regions (low pLDDT) should show dramatic motion while structured domains stay rigid
- If B-factor column is all zeros (some PDB files), fall back to pure contact-density method
- Glycan flexibility floor of 1.5 still respected

---

## Phase 2 — Core Improvements 🟡

*Moderate-effort changes that significantly improve physics fidelity and art quality.*

---

### Task 2.1: Sub-Pixel Gaussian Splatting

**Priority:** 🟡 | **Complexity:** M | **Dependencies:** None

**Rationale:** Currently, `accumulateChaos()` deposits intensity at integer pixel coordinates (`ix = px|0, iy = py|0`). This creates visible pixel-grid artifacts, especially at low zoom or with slow accumulation. Distributing each deposit as a small Gaussian kernel produces smoother, more photographic trails.

**Implementation:**

1. **In worker `accumulateChaos()`,** replace the single-pixel deposit block:

```javascript
// BEFORE (single pixel):
// const ix = px|0, iy = py|0;
// if (ix>=0 && ix<w && iy>=0 && iy<h) {
//     const idx = (iy*w+ix)*channels;
//     accumulationBuffer[idx] += val;
// }

// AFTER (3x3 Gaussian splat):
const ix = Math.floor(px), iy = Math.floor(py);
const fx = px - ix, fy = py - iy;  // fractional position [0, 1)

// Bilinear-inspired 2x2 splat (fast approximation of Gaussian)
// Weights based on fractional distance to each of 4 nearest pixels
const w00 = (1-fx)*(1-fy), w10 = fx*(1-fy);
const w01 = (1-fx)*fy,     w11 = fx*fy;

const splatPixels = [
    [ix,   iy,   w00],
    [ix+1, iy,   w10],
    [ix,   iy+1, w01],
    [ix+1, iy+1, w11]
];

for (let sp = 0; sp < 4; sp++) {
    const sx = splatPixels[sp][0], sy = splatPixels[sp][1];
    if (sx>=0 && sx<w && sy>=0 && sy<h) {
        const sIdx = (sy*w+sx)*channels;
        const sWeight = splatPixels[sp][2];
        if (useColor) {
            accumulationBuffer[sIdx]   += val*cr*sWeight;
            accumulationBuffer[sIdx+1] += val*cg*sWeight;
            accumulationBuffer[sIdx+2] += val*cb*sWeight;
        } else {
            accumulationBuffer[sIdx] += val*sWeight;
        }
    }
}
```

2. **Optional enhancement — true 3x3 Gaussian** for higher quality at the cost of 9 writes per deposit. Use a pre-computed 3x3 kernel and center it on the fractional position. Only enable this when `renderScale > 1` (high-res mode) to keep fast mode performant:

```javascript
// Pre-compute kernel once (sigma ≈ 0.7 pixels)
const GAUSS_3x3 = [
    0.0625, 0.125, 0.0625,
    0.125,  0.25,  0.125,
    0.0625, 0.125, 0.0625
];

for (let ky = -1; ky <= 1; ky++) {
    for (let kx = -1; kx <= 1; kx++) {
        const sx = ix+kx, sy = iy+ky;
        if (sx>=0 && sx<w && sy>=0 && sy<h) {
            const kIdx = (ky+1)*3+(kx+1);
            const sIdx = (sy*w+sx)*channels;
            accumulationBuffer[sIdx] += val * GAUSS_3x3[kIdx];
        }
    }
}
```

3. **Reduce step count to compensate** — since each deposit now covers multiple pixels, reduce the per-bond step count by ~50% to maintain similar performance:
```javascript
// Change: const steps = Math.max(1, Math.floor(dist));
// To:
const steps = Math.max(1, Math.floor(dist * 0.6));
```

**Acceptance Criteria:**
- At zoom levels < 20, trails appear smooth without visible pixel stepping
- No visible quality regression at high zoom levels
- Performance stays within 20% of current at "Fast" resolution
- Both density (1-channel) and chain/residue (3-channel) modes work correctly

---

### Task 2.2: Angular Potentials for Secondary Structure Preservation

**Priority:** 🟡 | **Complexity:** L | **Dependencies:** None

**Rationale:** The current model has no angular terms — only bond lengths and pairwise distance restraints. At high temperatures, alpha helices and beta sheets unfold unrealistically because nothing enforces the CA-CA-CA bond angle (which is ~90° in helices, ~120° in sheets). A simple 3-body cosine angle term keeps secondary structure intact while allowing interesting loop/domain motions.

**Implementation:**

1. **Add an `angles[]` array to the worker**, alongside `bonds[]` and `restraints[]`:
```javascript
let angles = []; // { i, j, k, theta0, kAngle }
```

2. **Build angle list in `parseAndLoad()`** (main thread), for every consecutive backbone triplet on the same chain:
```javascript
const angleList = [];
for (let i = 0; i < newAtoms.length - 2; i++) {
    if (newAtoms[i].isGlycan || newAtoms[i+1].isGlycan || newAtoms[i+2].isGlycan) continue;
    if (newAtoms[i].chain !== newAtoms[i+1].chain || newAtoms[i+1].chain !== newAtoms[i+2].chain) continue;

    // Compute native angle theta0
    const v1x = newAtoms[i].x - newAtoms[i+1].x;
    const v1y = newAtoms[i].y - newAtoms[i+1].y;
    const v1z = newAtoms[i].z - newAtoms[i+1].z;
    const v2x = newAtoms[i+2].x - newAtoms[i+1].x;
    const v2y = newAtoms[i+2].y - newAtoms[i+1].y;
    const v2z = newAtoms[i+2].z - newAtoms[i+1].z;

    const dot = v1x*v2x + v1y*v2y + v1z*v2z;
    const m1 = Math.hypot(v1x, v1y, v1z) || 1e-6;
    const m2 = Math.hypot(v2x, v2y, v2z) || 1e-6;
    const cosTheta = Math.max(-1, Math.min(1, dot / (m1 * m2)));
    const theta0 = Math.acos(cosTheta);

    angleList.push({ i, j: i+1, k: i+2, theta0 });
}
```

3. **Pass angles to worker** in the `init` message:
```javascript
worker.postMessage({ type: 'init', atoms, bonds: newBonds, restraints, angles: angleList, ... });
```

4. **In worker `onmessage` init handler,** store angles:
```javascript
angles = msg.angles || [];
```

5. **In worker `calcForces()`,** add angle force calculation after bond forces:
```javascript
// Angle forces — cosine harmonic: U = 0.5 * kAngle * (cos(theta) - cos(theta0))^2
const ANGLE_BASE_K = 40.0; // base stiffness, scaled by slider
const kAngle = ANGLE_BASE_K * bondScale;

for (let a = 0; a < angles.length; a++) {
    const { i, j, k, theta0 } = angles[a];

    const v1x = posX[i]-posX[j], v1y = posY[i]-posY[j], v1z = posZ[i]-posZ[j];
    const v2x = posX[k]-posX[j], v2y = posY[k]-posY[j], v2z = posZ[k]-posZ[j];

    const r1sq = v1x*v1x+v1y*v1y+v1z*v1z;
    const r2sq = v2x*v2x+v2y*v2y+v2z*v2z;
    const r1 = Math.sqrt(r1sq) || 1e-6;
    const r2 = Math.sqrt(r2sq) || 1e-6;

    const cosTheta = (v1x*v2x+v1y*v2y+v1z*v2z) / (r1*r2);
    const cosThetaClamped = Math.max(-0.999, Math.min(0.999, cosTheta));
    const cosTheta0 = Math.cos(theta0);

    // dU/d(cosTheta) = kAngle * (cosTheta - cosTheta0)
    const dUdCos = kAngle * (cosThetaClamped - cosTheta0);

    // Gradients of cos(theta) w.r.t. positions i, j, k
    const invR1 = 1.0/r1, invR2 = 1.0/r2;
    const invR1R2 = invR1 * invR2;

    // d(cosTheta)/d(xi) = (v2 - cosTheta*v1*r2/r1) / (r1*r2)
    // Simplified: force on atom i
    const f1x = dUdCos * (v2x*invR1R2 - cosThetaClamped*v1x*invR1*invR1);
    const f1y = dUdCos * (v2y*invR1R2 - cosThetaClamped*v1y*invR1*invR1);
    const f1z = dUdCos * (v2z*invR1R2 - cosThetaClamped*v1z*invR1*invR1);

    // Force on atom k
    const f3x = dUdCos * (v1x*invR1R2 - cosThetaClamped*v2x*invR2*invR2);
    const f3y = dUdCos * (v1y*invR1R2 - cosThetaClamped*v2y*invR2*invR2);
    const f3z = dUdCos * (v1z*invR1R2 - cosThetaClamped*v2z*invR2*invR2);

    // Force on atom j = -(F_i + F_k)  (Newton's 3rd law)
    forceX[i] -= f1x; forceY[i] -= f1y; forceZ[i] -= f1z;
    forceX[k] -= f3x; forceY[k] -= f3y; forceZ[k] -= f3z;
    forceX[j] += f1x+f3x; forceY[j] += f1y+f3y; forceZ[j] += f1z+f3z;
}
```

6. **Add `ANGLE_BASE_K` to CONF object** for tuning:
```javascript
const CONF = { ..., ANGLE_BASE_K: 40.0 };
```

**Acceptance Criteria:**
- Load `1UBQ` (ubiquitin) — alpha helix and beta sheet should maintain their native angles at moderate temperature (T=0.4)
- At very high temperature (T > 1.5), secondary structure should still partially unfold (not be completely rigid)
- Glycan atoms are excluded from angle calculations
- No NaN forces or simulation instability
- Performance: angle force loop should add < 5% overhead (angles.length ≈ nAtoms)

---

### Task 2.3: Residue-Specific WCA Excluded Volume

**Priority:** 🟡 | **Complexity:** M | **Dependencies:** None

**Rationale:** All beads currently use WCA_SIGMA = 1.0 regardless of residue size. Tryptophan (~186 Da) is physically 3x larger than glycine (~57 Da). Size-appropriate excluded volumes prevent unrealistic overlap and produce more natural-looking packing.

**Implementation:**

1. **Add per-atom sigma** to the atom data. In `parseAndLoad()`:
```javascript
// After computing simMass:
const sigma = 0.6 + 0.4 * Math.cbrt(mass / 186.0);  // range ~0.6 (GLY) to 1.0 (TRP)
newAtoms.push({ ..., sigma });
```

2. **Pass sigma to worker** in init message, stored as a Float32Array:
```javascript
// In worker init handler:
let sigmas = new Float32Array(nAtoms);
for (let i = 0; i < nAtoms; i++) {
    sigmas[i] = atoms[i].sigma || 1.0;
}
```

3. **Modify WCA force calculation** in `calcForces()` to use pair-averaged sigma:
```javascript
// Replace: const cutoffSq = CONF.WCA_CUTOFF_SQ;
// With per-pair calculation:
const si = sigmas[i], sj = sigmas[j];
const sigma_ij = (si + sj) * 0.5;
const sigma_ij_sq = sigma_ij * sigma_ij;
const cutoffSq_ij = sigma_ij_sq * 1.2599; // 2^(1/3) * sigma^2

if (r2 < cutoffSq_ij && r2 > 1e-5) {
    const r2i = sigma_ij_sq / r2;  // (sigma/r)^2
    const r6i = r2i * r2i * r2i;   // (sigma/r)^6
    const fS = Math.min((24*epsilon/r2) * sigma_ij_sq * (2*r6i*r6i - r6i), 500.0);
    // ... apply forces
}
```

4. **Update cell size** to accommodate the largest sigma:
```javascript
// In init handler, after computing sigmas:
let maxSigma = 1.0;
for (let i = 0; i < nAtoms; i++) if (sigmas[i] > maxSigma) maxSigma = sigmas[i];
WCA_CELL_SIZE_DYNAMIC = maxSigma * 1.3;
```

**Acceptance Criteria:**
- Glycine beads can pack closer together than tryptophan beads
- No overlap between large residues at equilibrium
- Cell size adapts to the largest residue in the structure
- Simulation remains stable (no NaN/Inf forces)

---

### Task 2.4: Secondary Structure Coloring Mode

**Priority:** 🟡 | **Complexity:** M | **Dependencies:** None

**Rationale:** Coloring by helix/sheet/loop is the most common coloring in structural biology and adds scientific value. PDB files contain HELIX/SHEET records; for mmCIF and AlphaFold files, a simple backbone-angle heuristic can assign secondary structure.

**Implementation:**

1. **Parse HELIX/SHEET records** in PDB parser:
```javascript
// In parseAndLoad PDB loop:
const ssMap = new Map();  // key: "chain_resNum" → value: 'H' | 'E' | 'C'

for (let l of lines) {
    if (l.startsWith('HELIX')) {
        const chain = l[19];
        const startRes = parseInt(l.substring(21, 25));
        const endRes = parseInt(l.substring(33, 37));
        for (let r = startRes; r <= endRes; r++) {
            ssMap.set(`${chain}_${r}`, 'H');
        }
    } else if (l.startsWith('SHEET')) {
        const chain = l[21];
        const startRes = parseInt(l.substring(22, 26));
        const endRes = parseInt(l.substring(33, 37));
        for (let r = startRes; r <= endRes; r++) {
            ssMap.set(`${chain}_${r}`, 'E');
        }
    }
}
```

2. **Assign SS to atoms:**
```javascript
for (let a of newAtoms) {
    a.ss = ssMap.get(`${a.chain}_${a.resNum}`) || 'C'; // C = coil/loop
}
```

3. **Fallback: backbone-angle heuristic** for files without HELIX/SHEET (mmCIF, AlphaFold):
```javascript
if (ssMap.size === 0 && newAtoms.length > 2) {
    for (let i = 1; i < newAtoms.length - 1; i++) {
        if (newAtoms[i].isGlycan || newAtoms[i].isNucleic) continue;
        if (newAtoms[i-1].chain !== newAtoms[i].chain || newAtoms[i].chain !== newAtoms[i+1].chain) continue;

        const v1x = newAtoms[i-1].x - newAtoms[i].x;
        const v1y = newAtoms[i-1].y - newAtoms[i].y;
        const v1z = newAtoms[i-1].z - newAtoms[i].z;
        const v2x = newAtoms[i+1].x - newAtoms[i].x;
        const v2y = newAtoms[i+1].y - newAtoms[i].y;
        const v2z = newAtoms[i+1].z - newAtoms[i].z;

        const dot = v1x*v2x + v1y*v2y + v1z*v2z;
        const m1 = Math.hypot(v1x, v1y, v1z) || 1e-6;
        const m2 = Math.hypot(v2x, v2y, v2z) || 1e-6;
        const angle = Math.acos(Math.max(-1, Math.min(1, dot / (m1*m2)))) * 180 / Math.PI;

        // CA-CA-CA angle: ~90° helix, ~120° sheet, other = coil
        if (angle < 105) newAtoms[i].ss = 'H';
        else if (angle < 140) newAtoms[i].ss = 'E';
        else newAtoms[i].ss = 'C';
    }
    // Smooth: require 3+ consecutive residues for assignment
    // (implementation: scan and reset isolated single-residue assignments)
}
```

4. **Add color mode** — add `<option value="ss">Secondary Structure</option>` to the `art-color-mode` dropdown.

5. **Define SS colors** alongside `RESIDUE_PROPERTIES`:
```javascript
const SS_COLORS = {
    'H': [0.94, 0.27, 0.51],  // Helix: magenta/red
    'E': [0.24, 0.60, 0.95],  // Sheet: blue
    'C': [0.75, 0.75, 0.75],  // Coil/loop: gray
};
```

6. **Update `rebuildColorArray()`** to handle the new mode:
```javascript
} else if (cMode === 'ss') {
    const c = SS_COLORS[atom.ss || 'C'];
    r = c[0]; g = c[1]; b = c[2];
    if (atom.isNucleic) { r=0.68; g=0.54; b=0.99; }
    if (atom.isGlycan) { r=0.97; g=0.62; b=0.07; }
}
```

7. **Show/hide SS sub-panel** in `onColorModeChange()` (follow the pattern of chain-panel/residue-panel):
```javascript
document.getElementById('ss-panel').style.display = (mode === 'ss') ? '' : 'none';
```

8. **Add SS sub-panel HTML** with background selector (same as residue panel).

**Acceptance Criteria:**
- Load `1UBQ` — helix is colored magenta, beta sheet blue, loops gray
- Load an AlphaFold model — angle-based SS assignment works when no HELIX/SHEET records
- Nucleic acids and glycans retain their own colors regardless of SS mode
- Background selector works correctly in SS mode
- 3D preview also reflects SS coloring

---

### Task 2.5: Ligand and Small Molecule Support

**Priority:** 🟡 | **Complexity:** M | **Dependencies:** None

**Rationale:** Important cofactors (heme in hemoglobin, ATP, FAD, drug molecules) are invisible since only CA/P/C1 are extracted. Adding them as single beads adds visual interest and scientific context.

**Implementation:**

1. **Define a cofactor list** (expand as needed):
```javascript
const COFACTOR_NAMES = new Set([
    'HEM', 'HEC',          // Heme variants
    'ATP', 'ADP', 'AMP',   // Adenine nucleotides
    'GTP', 'GDP', 'GMP',   // Guanine nucleotides
    'NAD', 'NAP', 'NDP',   // NAD/NADP
    'FAD', 'FMN',          // Flavin
    'COA', 'ACO',          // Coenzyme A
    'PLP',                  // Pyridoxal phosphate
    'SF4', 'FES',          // Iron-sulfur clusters
    'ZN', 'MG', 'CA', 'FE', 'MN', 'CU', 'CO', 'NI',  // Metal ions
    'CLR', 'PLM', 'OLA',  // Lipids
    'RET',                  // Retinal
]);

const COFACTOR_MASSES = {
    'HEM': 616, 'ATP': 507, 'ADP': 427, 'GTP': 523, 'NAD': 663,
    'FAD': 785, 'FMN': 456, 'COA': 767, 'PLP': 247,
    'SF4': 352, 'ZN': 65, 'MG': 24, 'CA': 40, 'FE': 56,
    'default': 200
};
```

2. **Extend the ATOM/HETATM parsing** to extract cofactors. Use a representative atom for each ligand residue (e.g., the first heavy atom, or a specific named atom):

```javascript
// After existing CA/P/C1 extraction, add cofactor check:
if (COFACTOR_NAMES.has(resName)) {
    // Use the first atom of each unique (chain, resNum, resName) as representative
    const cofKey = `${chain}_${resNum}_${resName}`;
    if (!cofactorSeen.has(cofKey)) {
        cofactorSeen.add(cofKey);
        const mass = COFACTOR_MASSES[resName] || COFACTOR_MASSES['default'];
        const simMass = mass / 110.0;
        newAtoms.push({
            x: x/4, y: y/4, z: z/4,
            isNucleic: false, isGlycan: false, isProtein: false,
            isCofactor: true,
            resName, simMass,
            visualWeight: Math.min(1.0, 0.4 + (mass / 800) * 0.6),
            chain, resNum
        });
    }
}
```

3. **Add `isCofactor` flag** to worker atom data:
```javascript
// In worker init:
let isCofactor = new Int8Array(nAtoms);
for (let i = 0; i < nAtoms; i++) {
    isCofactor[i] = atoms[i].isCofactor ? 1 : 0;
}
```

4. **Create proximity bonds** between cofactors and nearby protein/nucleic atoms:
```javascript
// After backbone bonds, before glycan logic:
const cofactorIndices = [];
newAtoms.forEach((a, i) => { if (a.isCofactor) cofactorIndices.push(i); });

for (let c of cofactorIndices) {
    let bestDist = Infinity, bestIdx = -1;
    for (let p = 0; p < newAtoms.length; p++) {
        if (p === c || newAtoms[p].isCofactor) continue;
        const d = Math.hypot(newAtoms[c].x-newAtoms[p].x, newAtoms[c].y-newAtoms[p].y, newAtoms[c].z-newAtoms[p].z);
        if (d < bestDist) { bestDist = d; bestIdx = p; }
    }
    if (bestIdx >= 0 && bestDist < 5.0) {
        newBonds.push({ i: c, j: bestIdx, visualWeight: 0.5, d: bestDist });
        uf.union(c, bestIdx);
    }
}
```

5. **Also create restraints** between cofactors and nearby atoms using existing `buildRestraintsSpatial` — the function already handles all non-glycan atoms, so cofactor atoms will automatically get restraints if close enough. Ensure cofactor atoms are not accidentally excluded.

6. **3D preview coloring** — add cofactor colors:
```javascript
const VIEW_COFACTOR_COLORS = [new THREE.Color(0xff6b35), new THREE.Color(0xff3864), new THREE.Color(0xffaa00)];
// In update3DColors:
if (atom.isCofactor) { base = VIEW_COFACTOR_COLORS; }
```

7. **Residue property coloring for cofactors:**
```javascript
// In rebuildColorArray:
if (atom.isCofactor) { r = 1.0; g = 0.42; b = 0.21; } // bright orange-red
```

8. **Add UI checkbox** to toggle cofactor display (similar to "Model Glycans"):
```html
<label class="text-xs text-gray-400 flex items-center gap-2 cursor-pointer hover:text-white transition-colors">
    <input type="checkbox" id="model-cofactors" class="w-3 h-3 accent-blue-500" checked />
    Model Cofactors
</label>
```

**Acceptance Criteria:**
- Load `2HHB` (hemoglobin) — four heme groups should appear as visible beads
- Load `1ATP` — ATP molecule appears anchored to the protein
- Cofactors are bonded to nearest protein atom and get restraints
- Cofactors have distinct coloring in all color modes
- Metal ions (small mass) appear as small beads
- Toggle checkbox correctly shows/hides cofactors on re-parse

---

## Phase 3 — Major Features 🔴

*High-impact features requiring significant development effort.*

---

### Task 3.1: Keyframe Animation Timeline

**Priority:** 🔴 | **Complexity:** XL | **Dependencies:** Task 1.2 (shareable URLs, for parameter encoding logic)

**Rationale:** Users currently can only record static simulation runs. A keyframe system that interpolates parameters over time (temperature, stiffness, rotation, zoom, theme) enables cinematic time-lapse sequences: e.g., start rigid → explode into chaos → slowly rotate → fade to different theme.

**Implementation:**

1. **Define keyframe data structure:**
```javascript
// Keyframe: snapshot of all animatable parameters at a given time
const ANIMATABLE_PARAMS = [
    'temp', 'k', 'speed', 'zoom', 'decay', 'gamma', 'softness',
    'intensity', 'jitter', 'bloom', 'depthfog', 'rotX', 'rotY', 'rotZ', 'panX', 'panY'
];

let keyframes = [];        // Array of { time: number (seconds), params: {} }
let timelinePlaying = false;
let timelineStartTime = 0;
let timelineDuration = 10; // seconds
```

2. **Add timeline UI** below the art panel or as a collapsible section in the sidebar:
```html
<div class="control-section" id="timeline-section">
    <div class="section-title">
        <svg class="w-3 h-3" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                  d="M12 8v4l3 3m6-3a9 9 0 11-18 0 9 9 0 0118 0z"></path>
        </svg>
        Animation Timeline
    </div>
    <div class="flex gap-2 mb-2">
        <button onclick="addKeyframe()" class="btn btn-secondary btn-xs flex-1">+ Keyframe</button>
        <button onclick="playTimeline()" id="btn-play-tl" class="btn btn-primary btn-xs flex-1">Play</button>
        <button onclick="clearTimeline()" class="btn btn-danger btn-xs">Clear</button>
    </div>
    <div class="flex items-center gap-2 mb-2">
        <span class="text-[10px] text-gray-500">Duration</span>
        <input type="number" id="tl-duration" min="1" max="120" value="10"
               class="w-16 text-[10px]" onchange="timelineDuration = parseInt(this.value)" />
        <span class="text-[10px] text-gray-500">sec</span>
    </div>
    <div id="keyframe-list" class="max-h-32 overflow-y-auto text-[10px] text-gray-400 font-mono"></div>
    <input type="range" id="tl-scrubber" min="0" max="100" value="0" step="0.1"
           oninput="scrubTimeline(this.value)" class="mt-2" />
</div>
```

3. **Implement keyframe capture:**
```javascript
window.addKeyframe = () => {
    const time = keyframes.length === 0 ? 0 :
        parseFloat(prompt('Time (seconds):', (timelineDuration).toFixed(1)));
    if (isNaN(time)) return;

    const params = {};
    ANIMATABLE_PARAMS.forEach(p => {
        const el = document.getElementById(paramToElementId(p));
        if (el) params[p] = parseFloat(el.value);
    });

    keyframes.push({ time, params });
    keyframes.sort((a, b) => a.time - b.time);
    renderKeyframeList();
};

function paramToElementId(param) {
    const map = {
        temp: 'temp-slider', k: 'k-restraint', speed: 'sim-speed',
        zoom: 'art-zoom-control', decay: 'art-decay', gamma: 'art-gamma',
        softness: 'art-softness', intensity: 'art-intensity', jitter: 'art-jitter',
        bloom: 'art-bloom', depthfog: 'art-depthfog',
        rotX: 'cam-rot-x', rotY: 'cam-rot-y', rotZ: 'cam-rot-z',
        panX: 'cam-pan-x', panY: 'cam-pan-y'
    };
    return map[param] || param;
}
```

4. **Implement timeline playback with interpolation:**
```javascript
window.playTimeline = () => {
    if (keyframes.length < 2) { alert('Need at least 2 keyframes'); return; }
    timelinePlaying = true;
    timelineStartTime = performance.now();
    // Ensure simulation is running
    if (!state.isRunning) window.toggleSim(true);
};

// Call this in the main tick() function:
function updateTimeline(now) {
    if (!timelinePlaying || keyframes.length < 2) return;
    const elapsed = (now - timelineStartTime) / 1000;
    const t = elapsed / timelineDuration;

    if (t >= 1.0) {
        timelinePlaying = false;
        document.getElementById('tl-scrubber').value = 100;
        return;
    }

    document.getElementById('tl-scrubber').value = t * 100;
    const currentTime = t * timelineDuration;

    // Find bracketing keyframes
    let kf0 = keyframes[0], kf1 = keyframes[keyframes.length - 1];
    for (let i = 0; i < keyframes.length - 1; i++) {
        if (keyframes[i].time <= currentTime && keyframes[i+1].time >= currentTime) {
            kf0 = keyframes[i]; kf1 = keyframes[i+1]; break;
        }
    }

    // Interpolation factor
    const range = (kf1.time - kf0.time) || 1;
    const alpha = (currentTime - kf0.time) / range;
    // Smooth interpolation (ease-in-out)
    const smooth = alpha * alpha * (3 - 2 * alpha);

    // Apply interpolated values to all sliders
    ANIMATABLE_PARAMS.forEach(p => {
        if (kf0.params[p] !== undefined && kf1.params[p] !== undefined) {
            const val = kf0.params[p] + (kf1.params[p] - kf0.params[p]) * smooth;
            const el = document.getElementById(paramToElementId(p));
            if (el) {
                el.value = val;
                // Trigger the oninput handler to update display values
                el.dispatchEvent(new Event('input'));
            }
        }
    });
}
```

5. **Add `updateTimeline(now)` call** in the main `tick()` function, before the worker tick:\
```javascript
// In tick(now):
updateTimeline(now);
```

6. **Implement scrubber for manual preview:**
```javascript
window.scrubTimeline = (pct) => {
    // Same as playback but at a specific point
    const t = pct / 100;
    const currentTime = t * timelineDuration;
    // ... same interpolation logic as playback, without auto-advancing
};
```

7. **Integrate with recording system** — when recording starts, optionally auto-play the timeline:
```javascript
// In startRecording, add option:
if (keyframes.length >= 2 && confirm('Play timeline during recording?')) {
    playTimeline();
}
```

**Acceptance Criteria:**
- User can add keyframes at specific times capturing current slider state
- Playback smoothly interpolates all parameters between keyframes
- Scrubber allows manual preview of any point in the timeline
- Recording mode can auto-play the timeline
- Works with all color modes and themes (theme switches can be handled as nearest-keyframe snap rather than interpolation)
- No interference with manual slider adjustments when timeline is not playing

---

### Task 3.2: NMR Ensemble / Multi-Model Visualization

**Priority:** 🔴 | **Complexity:** L | **Dependencies:** None

**Rationale:** NMR structures contain 10-40 conformational models. Currently only model 1 is used. Displaying all models as ghosted overlays reveals the conformational ensemble in a single image — a unique visualization not available in standard viewers.

**Implementation:**

1. **Extend parsing to capture all models.** In `parseAndLoad()` PDB parser, change the model logic:
```javascript
// Replace the current first-model-only logic:
let currentModel = 1;
let allModels = [[]]; // array of atom arrays, indexed by model number
let inModel = false;
let modelCount = 0;

for (let l of lines) {
    if (l.startsWith('MODEL')) {
        modelCount++;
        currentModel = parseInt(l.substring(6).trim()) || modelCount;
        if (!allModels[currentModel]) allModels[currentModel] = [];
        inModel = true;
        continue;
    }
    if (l.startsWith('ENDMDL')) {
        inModel = false;
        continue;
    }
    // ... parse ATOM/HETATM into allModels[currentModel] ...
}
```

2. **Add UI toggle** for ensemble mode:
```html
<label class="text-xs text-gray-400 flex items-center gap-2 cursor-pointer">
    <input type="checkbox" id="model-ensemble" class="w-3 h-3 accent-blue-500" />
    Show Ensemble (NMR)
</label>
<div class="slider-container" id="ensemble-alpha-container" style="display:none">
    <div class="slider-header">
        <span>Ghost Alpha</span>
        <span id="ensemble-alpha-val" class="slider-val">0.3</span>
    </div>
    <input type="range" id="ensemble-alpha" min="0.05" max="1.0" value="0.3" step="0.05" />
</div>
```

3. **Implement ensemble accumulation strategy.** Two approaches (implement option A first):

   **A. Static overlay:** After simulation initializes with model 1, project all other models' coordinates through the same rotation matrix and accumulate them as static ghost traces at reduced intensity:

```javascript
// New function in worker: accumulateEnsembleGhosts(models, alpha)
// Called once after init or when rotation changes
function accumulateEnsembleGhosts(ensembleModels, ghostAlpha) {
    for (let m = 0; m < ensembleModels.length; m++) {
        const model = ensembleModels[m]; // array of {x, y, z} per atom
        // Project and scatter like accumulateChaos() but using model coords
        // and intensity scaled by ghostAlpha
        for (let k = 0; k < bonds.length; k++) {
            // ... same scattering logic but with model[bonds[k].i] positions
            // ... intensity *= ghostAlpha
        }
    }
}
```

   **B. Dynamic ensemble:** Run N independent simulations in parallel (each model as starting coordinates) and blend their accumulation buffers. This is much more expensive but shows dynamic ensemble behavior.

4. **Pass ensemble data to worker** via a new message type:
```javascript
worker.postMessage({
    type: 'ensemble',
    models: allModels.slice(2).map(m => m.map(a => ({x: a.x, y: a.y, z: a.z})))
});
```

5. **Handle model detection:** Only enable ensemble UI when `modelCount > 1`:
```javascript
const hasEnsemble = modelCount > 1;
document.getElementById('model-ensemble').parentElement.style.display = hasEnsemble ? '' : 'none';
if (hasEnsemble) {
    document.getElementById('model-ensemble').parentElement.querySelector('span')
        .textContent = `Show Ensemble (${modelCount} models)`;
}
```

**Acceptance Criteria:**
- Load `2KOX` (NMR structure, 20 models) — with ensemble enabled, all 20 models appear as ghosted overlays
- Primary model (model 1) runs simulation normally; ghosts are static
- Ghost intensity adjustable via alpha slider
- Ensemble checkbox hidden for single-model structures (X-ray, AlphaFold)
- Performance: static overlay adds ghosts once per clear/rotation change, not per frame
- For mmCIF: parse `pdbx_PDB_model_num` column to separate models

---

### Task 3.3: Depth-of-Field Effect

**Priority:** 🔴 | **Complexity:** L | **Dependencies:** None

**Rationale:** The app already computes per-point depth (zNorm) for fog. Adding a focus-plane-based blur creates a photographic depth-of-field effect that adds significant visual quality. This is a post-processing effect on the 2D accumulation buffer.

**Implementation:**

1. **Add UI controls:**
```html
<div class="slider-container" title="Depth-of-field focus plane position">
    <div class="slider-header">
        <span>DoF Focus</span>
        <span id="dof-focus-val" class="slider-val">0.50</span>
    </div>
    <input type="range" id="art-dof-focus" min="0" max="1" value="0.5" step="0.01" />
</div>
<div class="slider-container" title="Depth-of-field blur strength">
    <div class="slider-header">
        <span>DoF Strength</span>
        <span id="dof-strength-val" class="slider-val">0.00</span>
    </div>
    <input type="range" id="art-dof-strength" min="0" max="1.0" value="0" step="0.01" />
</div>
```

2. **Strategy: Layered blur approach.** Split the accumulation into near/mid/far layers based on depth, blur the out-of-focus layers, then composite:

```javascript
// In renderDisplay(), after tone mapping but before bloom:
const dofStrength = parseFloat(document.getElementById('art-dof-strength')?.value || '0');
const dofFocus = parseFloat(document.getElementById('art-dof-focus')?.value || '0.5');

if (dofStrength > 0.01) {
    applyDepthOfField(artCtx, artWidth, artHeight, dofFocus, dofStrength);
}

function applyDepthOfField(ctx, w, h, focusPlane, strength) {
    // Create temporary canvases for near and far layers
    const nearCanvas = document.createElement('canvas');
    nearCanvas.width = w; nearCanvas.height = h;
    const nearCtx = nearCanvas.getContext('2d');

    const farCanvas = document.createElement('canvas');
    farCanvas.width = w; farCanvas.height = h;
    const farCtx = farCanvas.getContext('2d');

    // We need depth information per pixel — this requires the worker to
    // also send a depth buffer alongside the accumulation buffer.
    // ALTERNATIVE: approximate by blurring the entire image and blending
    // based on a radial/vertical gradient centered on the focus plane.

    // Simple approximation: apply variable blur based on vertical position
    // (assumes depth correlates roughly with screen Y after rotation)
    // This is a coarse but effective visual approximation.

    const maxBlur = Math.round(strength * w * 0.02);

    // Use the existing bloom canvas infrastructure for blur
    const tempCanvas = document.createElement('canvas');
    tempCanvas.width = w >> 1; tempCanvas.height = h >> 1;
    const tempCtx = tempCanvas.getContext('2d');

    ctx.save();
    // Draw blurred version
    tempCtx.filter = `blur(${maxBlur}px)`;
    tempCtx.drawImage(ctx.canvas, 0, 0, w >> 1, h >> 1);
    tempCtx.filter = 'none';

    // Create a gradient mask for blending
    const gradient = ctx.createLinearGradient(0, 0, 0, h);
    const focusY = focusPlane;
    const bandWidth = 0.15; // focus band width
    gradient.addColorStop(0, `rgba(0,0,0,${strength})`);
    gradient.addColorStop(Math.max(0, focusY - bandWidth), `rgba(0,0,0,${strength * 0.5})`);
    gradient.addColorStop(focusY, 'rgba(0,0,0,0)');
    gradient.addColorStop(Math.min(1, focusY + bandWidth), `rgba(0,0,0,${strength * 0.5})`);
    gradient.addColorStop(1, `rgba(0,0,0,${strength})`);

    // Composite: draw blurred version through gradient mask
    ctx.globalCompositeOperation = 'source-over';
    ctx.globalAlpha = 1.0;
    ctx.drawImage(tempCanvas, 0, 0, w, h);

    // Apply mask using 'destination-in' with gradient
    // (This is a simplification — for full quality, use a pixel-level depth buffer)
    ctx.restore();
}
```

3. **Better approach (requires worker change):** Have the worker maintain a parallel **depth buffer** (Float32Array, same size as accumulation buffer) that stores the average depth at each pixel. Pass this alongside the accumulation buffer. Then in `renderDisplay()`, use per-pixel depth to compute blur radius.

   In worker `accumulateChaos()`:
```javascript
// After: accumulationBuffer[idx] += val;
// Add:   depthBuffer[idx] = depthBuffer[idx] * 0.95 + zNorm * 0.05; // running average
```

   The depth buffer would need to be transferred alongside the accumulation buffer (or stored in a secondary channel).

**Acceptance Criteria:**
- DoF slider at 0 = no effect (default)
- Increasing DoF strength blurs regions away from focus plane
- Focus plane slider shifts the in-focus region
- Works with all themes and color modes
- Performance: < 5ms overhead per frame at fast resolution

---

### Task 3.4: SVG / Vector Export

**Priority:** 🔴 | **Complexity:** L | **Dependencies:** None

**Rationale:** Journal figures require vector output for infinite-resolution printing. Since we already have 2D projected coordinates, we can export the current frame's bond topology as SVG paths with theme-appropriate colors.

**Implementation:**

1. **Add export button** in Data Actions section:
```html
<button onclick="exportSVG()" class="btn btn-secondary text-xs">Export SVG</button>
```

2. **Implement SVG generation:**
```javascript
window.exportSVG = () => {
    if (!window.globalAtoms || !window.globalBonds) {
        alert('Load a structure first');
        return;
    }

    const atoms = window.globalAtoms;
    const bonds = window.globalBonds;
    const w = artWidth, h = artHeight;
    const theme = ui.theme.value;
    const bg = getThemeBackground(theme);
    const zoom = parseFloat(ui.zoom.value) * renderScale;
    const panX = parseFloat(ui.panX.value) * renderScale;
    const panY = parseFloat(ui.panY.value) * renderScale;

    // Build rotation matrix (same as worker accumulateChaos)
    const radX = (parseFloat(ui.rotX.value)||0)*Math.PI/180;
    const radY = (parseFloat(ui.rotY.value)||0)*Math.PI/180;
    const radZ = (parseFloat(ui.rotZ.value)||0)*Math.PI/180;
    const cxr=Math.cos(radX),sxr=Math.sin(radX);
    const cyr=Math.cos(radY),syr=Math.sin(radY);
    const czr=Math.cos(radZ),szr=Math.sin(radZ);
    const m00=cyr*czr, m01=czr*sxr*syr-cxr*szr, m02=cxr*czr*syr+sxr*szr;
    const m10=cyr*szr, m11=cxr*czr+sxr*syr*szr, m12=cxr*syr*szr-czr*sxr;

    // Project atoms to 2D
    const projected = atoms.map(a => ({
        x: (m00*a.x + m01*a.y + m02*a.z) * zoom + w/2 + panX,
        y: (m10*a.x + m11*a.y + m12*a.z) * zoom + h/2 + panY,
    }));

    // Generate SVG
    let svg = `<?xml version="1.0" encoding="UTF-8"?>
<svg xmlns="http://www.w3.org/2000/svg" width="${w}" height="${h}" viewBox="0 0 ${w} ${h}">
<rect width="100%" height="100%" fill="rgb(${bg.r},${bg.g},${bg.b})" />
<g opacity="0.8">
`;

    // Get stroke color from theme
    const midColor = applyTheme(theme, 0.7, bg);
    const strokeColor = `rgb(${Math.round(midColor.r)},${Math.round(midColor.g)},${Math.round(midColor.b)})`;

    // Draw bonds as paths
    for (const bond of bonds) {
        const p1 = projected[bond.i], p2 = projected[bond.j];
        const vw = bond.visualWeight || 0.5;
        const strokeWidth = Math.max(0.3, vw * 2);
        svg += `  <line x1="${p1.x.toFixed(1)}" y1="${p1.y.toFixed(1)}" x2="${p2.x.toFixed(1)}" y2="${p2.y.toFixed(1)}" stroke="${strokeColor}" stroke-width="${strokeWidth}" stroke-linecap="round" opacity="${vw}" />\n`;
    }

    // Optionally draw atom circles
    for (let i = 0; i < atoms.length; i++) {
        const p = projected[i];
        const r = Math.max(0.5, atoms[i].visualWeight * 3);
        svg += `  <circle cx="${p.x.toFixed(1)}" cy="${p.y.toFixed(1)}" r="${r}" fill="${strokeColor}" opacity="0.6" />\n`;
    }

    svg += `</g>\n</svg>`;

    // Download
    const blob = new Blob([svg], { type: 'image/svg+xml' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.download = `chaos_${currentPdbId}_${theme}.svg`;
    a.href = url;
    a.click();
    URL.revokeObjectURL(url);
};
```

3. **Enhanced version — chain coloring in SVG:**
```javascript
// If color mode is 'chain', use per-bond colors:
const cMode = document.getElementById('art-color-mode')?.value || 'density';
if (cMode === 'chain') {
    for (const bond of bonds) {
        const hex1 = window.chainColors[atoms[bond.i].chain] || '#93c5fd';
        const hex2 = window.chainColors[atoms[bond.j].chain] || '#93c5fd';
        // Use hex1 for the line (or create a gradient between hex1 and hex2)
        svg += `  <line ... stroke="${hex1}" ... />\n`;
    }
}
```

4. **Add SVG metadata:**
```javascript
// Add comment header to SVG
svg = `<!-- ProteinCHAOS SVG Export | ${currentPdbId} | Theme: ${theme} -->\n` + svg;
```

**Acceptance Criteria:**
- SVG file opens correctly in Illustrator, Inkscape, and browsers
- Background color matches the selected theme
- Bond stroke colors match the active color mode
- Bond thickness reflects visual weight
- File size reasonable (< 1MB for most structures)
- Coordinates match the current camera orientation (rotation, zoom, pan)

---

## Phase 4 — Architecture Upgrades 🔴

*Fundamental architecture changes that unlock order-of-magnitude performance or capability improvements. These require significant refactoring and should be done after Phases 1–3 are stable.*

---

### Task 4.1: WebGL/WebGPU Accumulation Shader

**Priority:** 🔴 | **Complexity:** XL | **Dependencies:** None (but benefits from Task 2.1 being done first to understand the splatting model)

**Rationale:** The single biggest bottleneck is CPU-based point scattering in the worker's `accumulateChaos()`. Moving this to a GPU compute shader or WebGL fragment shader would give 10–100x speedup for the rendering pass, enabling real-time 4K, Gaussian splatting, and volumetric effects.

**Implementation Strategy:**

1. **Choose API:** WebGPU is ideal (compute shaders) but limited browser support. WebGL2 is universally supported and sufficient via transform feedback or texture ping-pong. **Recommend: WebGL2 first, WebGPU as optional upgrade.**

2. **Architecture change:**
```
Current:  Worker → accumulateChaos() on CPU → transfer Float32Array → Main renders ImageData
Proposed: Worker → physics only → positions Float32Array → Main → GPU accumulation + tone mapping
```

3. **WebGL2 implementation sketch:**

   a. **Create a second WebGL2 context** on the art canvas (or a separate canvas composited over):
```javascript
const gl = artCanvas.getContext('webgl2', { preserveDrawingBuffer: true });
```

   b. **Upload positions as a texture** (nAtoms × 1, RGBA32F):**
```javascript
const posTexture = gl.createTexture();
// Each texel = (x, y, z, visualWeight) for one atom
// Upload: gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA32F, nAtoms, 1, 0, gl.RGBA, gl.FLOAT, posData);
```

   c. **Accumulation shader (fragment shader):** For each bond, draw a line primitive (or a quad strip) that scatters density. The shader reads endpoint positions from the texture and outputs to an accumulation FBO (floating-point framebuffer):

```glsl
// Vertex shader: emit points along each bond
uniform sampler2D u_positions;
uniform float u_zoom, u_panX, u_panY;
uniform mat3 u_rotMatrix;
uniform float u_intensity, u_jitter;

// Bond index → atom indices passed via attributes
attribute float a_bondParam; // t parameter [0, 1]
attribute vec2 a_atomIndices; // (i, j)

void main() {
    vec3 posI = texelFetch(u_positions, ivec2(int(a_atomIndices.x), 0), 0).xyz;
    vec3 posJ = texelFetch(u_positions, ivec2(int(a_atomIndices.y), 0), 0).xyz;
    vec3 pos3D = mix(posI, posJ, a_bondParam);
    vec3 rotated = u_rotMatrix * pos3D;
    vec2 screen = rotated.xy * u_zoom + vec2(u_panX, u_panY);
    gl_Position = vec4(screen / vec2(u_width, u_height) * 2.0 - 1.0, 0.0, 1.0);
    gl_PointSize = 1.0;
}

// Fragment shader: accumulate into floating-point FBO
void main() {
    gl_FragColor = vec4(u_intensity, 0.0, 0.0, 1.0);
}
```

   d. **Tone mapping shader:** Read from accumulation FBO, apply the same log1p → power → ACES → theme color pipeline:
```glsl
uniform sampler2D u_accumBuffer;
uniform float u_displayMax, u_sharpness, u_threshold;

vec3 acesToneMap(vec3 x) {
    return (x * (2.51 * x + 0.03)) / (x * (2.43 * x + 0.59) + 0.14);
}

void main() {
    float val = texture2D(u_accumBuffer, v_uv).r;
    float logC = log(1.0 + val) / log(1.0 + u_displayMax);
    float powered = pow(logC, u_sharpness);
    // ... threshold, ACES, theme color ...\n    gl_FragColor = vec4(color, 1.0);
}
```

   e. **Decay:** Multiply accumulation FBO by decay factor each frame using a fullscreen quad shader with `glBlendFunc(gl.ZERO, gl.SRC_COLOR)`.

4. **Fallback:** Keep the current CPU path as a fallback for browsers without WebGL2 float texture support. Feature-detect:
```javascript
const ext = gl.getExtension('EXT_color_buffer_float');
const useGPU = !!ext;
```

5. **Data flow change:** The worker no longer maintains `accumulationBuffer`. Instead, it sends only positions to main, and main handles accumulation on GPU:
```javascript
// Worker: remove accumulateChaos() and fadeAccumulationBuffer()
// Worker: only sends positions in 'update' message (no buffer transfer)
// Main: uploads positions to GPU, runs accumulation + tone mapping shaders
```

**Acceptance Criteria:**
- Visual output matches CPU version (same accumulation, tone mapping, themes)
- 60fps at 4K resolution on mid-range GPU
- CPU fallback works when WebGL2 is unavailable
- All color modes (density, chain, residue) work on GPU path
- Bloom and grain still applied as post-process

---

### Task 4.2: WebAssembly Physics Engine

**Priority:** 🔴 | **Complexity:** XL | **Dependencies:** None

**Rationale:** The hot loops (`integrate()`, `calcForces()`, `buildWCACells()`) are pure numerical computation ideal for WebAssembly. A Rust or C implementation compiled to WASM would give 2–5x speedup, making structures with >5000 residues responsive.

**Implementation Strategy:**

1. **Create a Rust crate** with the physics engine:
```
protein-chaos-physics/
├── Cargo.toml
├── src/
│   ├── lib.rs          // WASM entry points
│   ├── integrator.rs   // Langevin dynamics
│   ├── forces.rs       // Bond, restraint, WCA, glycan forces
│   ├── cells.rs        // Spatial hashing
│   └── noise.rs        // Box-Muller cache
```

2. **Export functions via wasm-bindgen:**
```rust
#[wasm_bindgen]
pub fn init_simulation(atoms_json: &str, bonds_json: &str, restraints_json: &str) -> bool;

#[wasm_bindgen]
pub fn step(n_substeps: u32, temp: f32, k: f32, speed: f32) -> *const f32;
// Returns pointer to positions array in WASM linear memory

#[wasm_bindgen]
pub fn set_params(temp: f32, k: f32, speed: f32, /* ... */);

#[wasm_bindgen]
pub fn get_positions_ptr() -> *const f32;
pub fn get_positions_len() -> usize;
```

3. **Integration with existing worker:**
```javascript
// In worker, replace JS physics with WASM calls:
let wasmModule = null;

// Load WASM (fetched as base64-encoded string embedded in the HTML, or from CDN)
async function initWASM() {
    const wasmBytes = await fetch('physics.wasm').then(r => r.arrayBuffer());
    const { instance } = await WebAssembly.instantiate(wasmBytes, {});
    wasmModule = instance.exports;
}

// In integrate():
if (wasmModule) {
    wasmModule.step(substeps, params.temp, params.k, params.speed);
    // Read positions from WASM memory:
    const ptr = wasmModule.get_positions_ptr();
    const len = wasmModule.get_positions_len();
    const wasmPositions = new Float32Array(wasmModule.memory.buffer, ptr, len);
    posX.set(wasmPositions.subarray(0, nAtoms));
    posY.set(wasmPositions.subarray(nAtoms, nAtoms*2));
    posZ.set(wasmPositions.subarray(nAtoms*2, nAtoms*3));
} else {
    // Fallback to JS physics
    for (let i = 0; i < substeps; i++) integrate();
}
```

4. **Build pipeline:** Use `wasm-pack` to build, then embed the `.wasm` binary as a base64 data URI in the HTML file to maintain the single-file constraint:
```javascript
const WASM_B64 = 'AGFzbQEAAAAB...'; // base64-encoded .wasm
const wasmBytes = Uint8Array.from(atob(WASM_B64), c => c.charCodeAt(0));
```

5. **Fallback:** The existing JS physics code remains as the fallback when WASM fails to load.

**Acceptance Criteria:**
- Physics results match JS implementation (within floating-point tolerance)
- 2x+ speedup for structures with N > 2000 atoms
- WASM binary < 100KB (gzipped)
- Graceful fallback to JS when WASM unavailable
- Single-file deployment maintained via base64 embedding

---

### Task 4.3: Flat-Array Spatial Hashing

**Priority:** 🟡 | **Complexity:** M | **Dependencies:** None

**Rationale:** The WCA cell list uses JavaScript `Map` with computed keys. For large systems, a flat `Int32Array`-based open-addressing hash table is significantly faster due to cache locality and avoiding GC pressure from Map operations.

**Implementation:**

1. **Replace `wcaCells` Map** with a flat hash table in the worker:

```javascript
// Hash table: parallel arrays for keys and value-lists
const HASH_TABLE_SIZE = 65536; // power of 2
const HASH_EMPTY = -1;
let hashKeys = new Int32Array(HASH_TABLE_SIZE).fill(HASH_EMPTY);
let hashHeads = new Int32Array(HASH_TABLE_SIZE).fill(-1);  // head of linked list
let hashNext = new Int32Array(0);    // per-atom next pointer
let hashBucket = new Int32Array(0);  // per-atom bucket assignment

function hashKey(cx, cy, cz) {
    // FNV-1a-inspired hash
    let h = 2166136261;
    h = (h ^ (cx + 512)) * 16777619;
    h = (h ^ (cy + 512)) * 16777619;
    h = (h ^ (cz + 512)) * 16777619;
    return (h >>> 0) & (HASH_TABLE_SIZE - 1);
}

function buildCellsFlat() {
    hashKeys.fill(HASH_EMPTY);
    hashHeads.fill(-1);
    if (hashNext.length < nAtoms) {
        hashNext = new Int32Array(nAtoms);
    }
    hashNext.fill(-1);

    for (let i = 0; i < nAtoms; i++) {
        const cx = Math.floor(posX[i] / WCA_CELL_SIZE);
        const cy = Math.floor(posY[i] / WCA_CELL_SIZE);
        const cz = Math.floor(posZ[i] / WCA_CELL_SIZE);
        const key = wcaKey(cx, cy, cz); // original integer key
        let bucket = hashKey(cx, cy, cz);

        // Linear probing to find or create bucket
        while (hashKeys[bucket] !== HASH_EMPTY && hashKeys[bucket] !== key) {
            bucket = (bucket + 1) & (HASH_TABLE_SIZE - 1);
        }
        hashKeys[bucket] = key;

        // Prepend atom to linked list for this bucket
        hashNext[i] = hashHeads[bucket];
        hashHeads[bucket] = i;
    }
}

// Iteration: for a cell at (cx, cy, cz):
function getCellHead(cx, cy, cz) {
    const key = wcaKey(cx, cy, cz);
    let bucket = hashKey(cx, cy, cz);
    while (hashKeys[bucket] !== HASH_EMPTY) {
        if (hashKeys[bucket] === key) return hashHeads[bucket];
        bucket = (bucket + 1) & (HASH_TABLE_SIZE - 1);
    }
    return -1; // empty cell
}

// Traverse linked list:
// for (let j = getCellHead(cx, cy, cz); j !== -1; j = hashNext[j]) { ... }
```

2. **Update all WCA/glycan force loops** to use the new iteration pattern.

3. **Replace `buildWCACells()` call** with `buildCellsFlat()`.

**Acceptance Criteria:**
- Identical force calculation results (verify with a test structure)
- Measurable speedup for N > 3000 atoms (benchmark before/after)
- No memory leaks (flat arrays are pre-allocated)
- Hash table size auto-scales for very large structures

---

### Task 4.4: Biological Assembly Support

**Priority:** 🔴 | **Complexity:** L | **Dependencies:** Task 2.5 (ligand support helps with assembly cofactors)

**Rationale:** Many iconic structures (viral capsids, ion channels, ribosomes) require symmetry operations to build the biological unit. The asymmetric unit alone often shows only one subunit of what should be a multi-chain complex.

**Implementation:**

1. **Parse REMARK 350** from PDB files:
```javascript
// In parseAndLoad PDB parser:
const biomt = []; // Array of 3x4 transformation matrices
let currentOper = null;

for (let l of lines) {
    if (l.startsWith('REMARK 350') && l.includes('BIOMT')) {
        const row = parseInt(l.substring(18, 19)); // 1, 2, or 3
        const operNum = parseInt(l.substring(19, 23));
        const m1 = parseFloat(l.substring(23, 33));
        const m2 = parseFloat(l.substring(33, 43));
        const m3 = parseFloat(l.substring(43, 53));
        const t  = parseFloat(l.substring(53, 68));

        if (row === 1) {
            currentOper = { m: new Float64Array(12) };
            biomt.push(currentOper);
        }
        if (currentOper) {
            const base = (row - 1) * 4;
            currentOper.m[base] = m1;
            currentOper.m[base+1] = m2;
            currentOper.m[base+2] = m3;
            currentOper.m[base+3] = t;
        }
    }
}
```

2. **Parse mmCIF** `_pdbx_struct_oper_list` and `_pdbx_struct_assembly_gen`:
```javascript
const operList = loops.get('_pdbx_struct_oper_list');
if (operList) {
    for (const row of operList.rows) {
        const m = new Float64Array(12);
        m[0] = parseFloat(row['matrix[1][1]']); m[1] = parseFloat(row['matrix[1][2]']);
        m[2] = parseFloat(row['matrix[1][3]']); m[3] = parseFloat(row['vector[1]']);
        // ... rows 2 and 3
        biomt.push({ m, id: row['id'] });
    }
}
```

3. **Add UI toggle:**
```html
<label class="text-xs text-gray-400 flex items-center gap-2 cursor-pointer">
    <input type="checkbox" id="model-assembly" class="w-3 h-3 accent-blue-500" />
    Biological Assembly
</label>
```

4. **Apply transformations** to duplicate atoms after initial parsing:
```javascript
if (document.getElementById('model-assembly').checked && biomt.length > 1) {
    const originalAtoms = [...newAtoms];
    const originalBonds = [...newBonds];

    for (let op = 1; op < biomt.length; op++) { // skip identity (op 0)
        const m = biomt[op].m;
        const offset = newAtoms.length;

        for (const a of originalAtoms) {
            const nx = m[0]*a.x*4 + m[1]*a.y*4 + m[2]*a.z*4 + m[3];
            const ny = m[4]*a.x*4 + m[5]*a.y*4 + m[6]*a.z*4 + m[7];
            const nz = m[8]*a.x*4 + m[9]*a.y*4 + m[10]*a.z*4 + m[11];
            newAtoms.push({
                ...a,
                x: nx/4, y: ny/4, z: nz/4,
                chain: a.chain + '_' + op  // unique chain ID
            });
        }

        for (const b of originalBonds) {
            newBonds.push({
                i: b.i + offset,
                j: b.j + offset,
                visualWeight: b.visualWeight,
                d: b.d
            });
        }
    }

    // Re-center after assembly
    let cx=0, cy=0, cz=0;
    for (let a of newAtoms) { cx+=a.x; cy+=a.y; cz+=a.z; }
    cx/=newAtoms.length; cy/=newAtoms.length; cz/=newAtoms.length;
    for (let a of newAtoms) { a.x-=cx; a.y-=cy; a.z-=cz; }
}
```

5. **Warning for large assemblies:**
```javascript
const totalAtoms = originalAtoms.length * biomt.length;
if (totalAtoms > 10000) {
    if (!confirm(`Biological assembly has ${totalAtoms} residues. This may be slow. Continue?`)) {
        // Revert to asymmetric unit
    }
}
```

**Acceptance Criteria:**
- Load `1BNA` (B-DNA, assembly = unit cell) — full double helix appears when assembly enabled
- Load `4HHB` (hemoglobin tetramer) — all 4 subunits visible
- Load `1A34` (viral capsid, huge assembly) — warning dialog appears for large assemblies
- Each symmetry copy gets a unique chain ID for proper chain coloring
- Bonds within each copy are preserved; inter-copy restraints are auto-generated by `buildRestraintsSpatial`
- Toggle checkbox re-parses the structure
- Assembly off (default) retains current behavior exactly

---

## Dependency Graph

```
Phase 1 (Independent — can be done in any order):
  1.1 SSBOND ──────────────────────┐
  1.2 Share URLs ──────────────────┤
  1.3 Progress Indicator ──────────┤  All independent
  1.4 Touch Gestures ──────────────┤
  1.5 B-factor Flexibility ────────┘

Phase 2 (Independent of Phase 1):
  2.1 Sub-pixel Splatting ─────────┐
  2.2 Angular Potentials ──────────┤
  2.3 Residue-specific WCA ────────┤  All independent
  2.4 SS Coloring Mode ────────────┤
  2.5 Ligand Support ──────────────┘

Phase 3 (Some dependencies):
  3.1 Keyframe Timeline ──── depends on 1.2 (parameter encoding)
  3.2 NMR Ensemble ────────── independent
  3.3 Depth of Field ──────── independent
  3.4 SVG Export ──────────── independent

Phase 4 (Architecture):
  4.1 WebGL Accumulation ──── independent (replaces worker accumulation)
  4.2 WASM Physics ────────── independent (replaces worker physics)
  4.3 Flat-array Hashing ──── independent (worker optimization)
  4.4 Biological Assembly ─── benefits from 2.5 (ligands in assemblies)
```

## Recommended Implementation Order

For maximum impact with minimum risk:

```
Sprint 1 (Quick wins):     1.3 → 1.1 → 1.2 → 1.5 → 1.4
Sprint 2 (Art quality):    2.1 → 2.4 → 2.5
Sprint 3 (Physics):        2.2 → 2.3 → 4.3
Sprint 4 (Major features): 3.4 → 3.1 → 3.2
Sprint 5 (Advanced):       3.3 → 4.4
Sprint 6 (Architecture):   4.1 → 4.2
```

## Testing Checklist

For each task, verify against these test structures:

| PDB ID | Features to Test |
|--------|-----------------|
| `1UBQ` | Small protein, helix+sheet, B-factors available |
| `2HHB` | Multi-chain, heme cofactors, SSBOND |
| `1IGT` | Antibody, many disulfide bonds, glycans |
| `2KOX` | NMR ensemble (20 models) |
| `1BNA` | B-form DNA, biological assembly |
| `4V6X` | Ribosome (very large, performance test) |
| `AF-Q5VSL9` | AlphaFold model, pLDDT, disordered regions |
| `6VYB` | SARS-CoV-2 spike, heavy glycosylation |
| `1ATP` | ATP cofactor bound to protein |

**General regression tests after each change:**
- [ ] Simulation starts and runs without NaN/Inf
- [ ] 3D preview shows correct structure
- [ ] Art canvas accumulates properly in all 3 color modes
- [ ] Download HD produces correct image
- [ ] All 20 themes render without errors
- [ ] Mobile layout not broken
- [ ] Recording (free + boomerang) produces valid video
- [ ] Browser console shows no errors or warnings
- [ ] Performance: frame time < 20ms at "Fast" resolution for structures < 1000 residues
