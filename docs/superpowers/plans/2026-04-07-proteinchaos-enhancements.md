# ProteinCHAOS Enhancements Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Upgrade ProteinCHAOS v1.1 to v1.2 with faster physics (cell-list WCA, noise cache, matrix precompute, transferable buffers), better rendering (ACES tonemapping, bloom, value-noise grain, depth cueing), per-chain color pickers, new palettes, residue-property coloring, auto-spin, and UX fixes — all within the single `index.html` file.

**Architecture:** All code lives in `index.html`. The Web Worker is embedded as a string (`workerCode`, lines 721–1190). Main-thread code runs from line 1192 onward. No build step — edit and serve with `python3 -m http.server 8080`. Worker and main thread communicate via `postMessage`; buffer transfers use transferables to avoid GC-heavy allocations.

**Tech Stack:** Vanilla JS (ES2020), Three.js r0.160 (CDN), Tailwind CSS v4 (CDN), Web Worker (Blob URL), Canvas 2D API, MediaRecorder API.

---

## Verification setup

For every task: open `http://localhost:8080`, load PDB `2HHB` (small hemoglobin, 4 chains), run simulation 10 seconds, observe the described effect. Console must be error-free.

```bash
cd /home/dzyla/ProteinCHAOS && python3 -m http.server 8080
```

---

## Task 1: Worker — Noise cache, cell-list WCA, substep cap

**Files:**
- Modify: `index.html` — worker string, lines 722–1190

This task replaces three slow paths inside the worker code string:
1. Box-Muller per call → pre-baked Gaussian cache
2. O(N²) WCA loop (disabled above 1500 atoms) → O(N) cell-list (always enabled)
3. Substep cap 20 → 40

- [ ] **Step 1: Replace global state + `randNormal` at top of worker string**

Find the block starting at line 740 (`const FADER_BLOCK_SIZE`). Replace from `const FADER_BLOCK_SIZE` through `let forceHighFPS = false;` with the version below. The key additions: `NOISE_SIZE`, `noiseCache`, `noiseCursor`, `wcaCells`, `WCA_CELL_SIZE`, and the new `initNoiseCache` + `buildWCACells` functions, plus the rewritten `randNormal`.

```js
      const FADER_BLOCK_SIZE = 524288;
      let sentPositions = null;
      let sentBuffer = null;

      const NOISE_SIZE = 65536;
      let noiseCache = null;
      let noiseCursor = 0;
      const WCA_CELL_SIZE = 1.3;
      let wcaCells = null;

      let nAtoms = 0;
      let posX, posY, posZ;
      let refX, refY, refZ;
      let velX, velY, velZ;
      let forceX, forceY, forceZ;
      let masses;
      let invMasses;
      let isNucleic;
      let isGlycan;
      let bonds = [];
      let restraints = [];
      let drawRestraints = [];
      let accumulationBuffer = null;
      let atomColors = null;
      let colorMode = 'density';

      let artWidth = 0, artHeight = 0;
      let centerX = 0, centerY = 0, centerZ = 0;
      let stepCount = 0;
      let isRunning = false;
      let fadeCursor = 0;
      let needsFrame = false;
      let simTime = 0.0;
      let frameCounter = 0;
      let displayFrameSkip = 2;

      let params = {
          temp: 0.4, k: 20.0, speed: 1.0, zoom: 35.0, jitter: 0.8,
          intensity: 40, source: 'trace', decay: 0.98,
          rotX: 0, rotY: 0, rotZ: 0, panX: 0, panY: 0,
          colorMode: 'density'
      };

      let forceHighFPS = false;

      function initNoiseCache() {
          noiseCache = new Float32Array(NOISE_SIZE);
          for (let i = 0; i < NOISE_SIZE - 1; i += 2) {
              let u = 0, v = 0;
              while (u === 0) u = Math.random();
              while (v === 0) v = Math.random();
              const mag = Math.sqrt(-2.0 * Math.log(u));
              noiseCache[i]   = mag * Math.cos(2.0 * Math.PI * v);
              noiseCache[i+1] = mag * Math.sin(2.0 * Math.PI * v);
          }
          noiseCursor = 0;
      }

      function randNormal() {
          if (noiseCursor >= NOISE_SIZE) noiseCursor = 0;
          return noiseCache[noiseCursor++];
      }

      function buildWCACells() {
          if (nAtoms === 0) { wcaCells = null; return; }
          wcaCells = new Map();
          for (let i = 0; i < nAtoms; i++) {
              const cx = Math.floor(posX[i] / WCA_CELL_SIZE);
              const cy = Math.floor(posY[i] / WCA_CELL_SIZE);
              const cz = Math.floor(posZ[i] / WCA_CELL_SIZE);
              const key = cx + ',' + cy + ',' + cz;
              let cell = wcaCells.get(key);
              if (!cell) { cell = []; wcaCells.set(key, cell); }
              cell.push(i);
          }
      }
```

- [ ] **Step 2: Replace O(N²) WCA section in `calcForces`**

Find the block (lines ~951–971):
```js
          if (nAtoms > 1500) return;
          const cutoffSq = CONF.WCA_CUTOFF_SQ;
          const epsilon = CONF.WCA_EPSILON;
          for (let i = 0; i < nAtoms; i++) {
              for (let j = i + 1; j < nAtoms; j++) {
```
...through the closing `}` of that double loop. Replace entirely with:

```js
          // WCA excluded-volume via cell list (O(N), works at all system sizes)
          if (wcaCells) {
              const cutoffSq = CONF.WCA_CUTOFF_SQ;
              const epsilon  = CONF.WCA_EPSILON;
              for (let i = 0; i < nAtoms; i++) {
                  const cx = Math.floor(posX[i] / WCA_CELL_SIZE);
                  const cy = Math.floor(posY[i] / WCA_CELL_SIZE);
                  const cz = Math.floor(posZ[i] / WCA_CELL_SIZE);
                  for (let dx = -1; dx <= 1; dx++) {
                      for (let dy = -1; dy <= 1; dy++) {
                          for (let dz = -1; dz <= 1; dz++) {
                              const cell = wcaCells.get((cx+dx)+','+(cy+dy)+','+(cz+dz));
                              if (!cell) continue;
                              for (let ki = 0; ki < cell.length; ki++) {
                                  const j = cell[ki];
                                  if (j <= i || Math.abs(i - j) <= 1) continue;
                                  const ddx = posX[i] - posX[j];
                                  const ddy = posY[i] - posY[j];
                                  const ddz = posZ[i] - posZ[j];
                                  const r2 = ddx*ddx + ddy*ddy + ddz*ddz;
                                  if (r2 < cutoffSq && r2 > 1e-5) {
                                      const r2i = 1.0 / r2;
                                      const r6i = r2i * r2i * r2i;
                                      const fS = Math.min((24*epsilon*r2i)*(2*r6i*r6i - r6i), 500.0);
                                      forceX[i] += ddx*fS; forceY[i] += ddy*fS; forceZ[i] += ddz*fS;
                                      forceX[j] -= ddx*fS; forceY[j] -= ddy*fS; forceZ[j] -= ddz*fS;
                                  }
                              }
                          }
                      }
                  }
              }
          }
```

- [ ] **Step 3: Update the `init` message handler to call both init functions**

Inside the `if (msg.type === 'init')` block, after `stepCount = 0; needsFrame = true; simTime = 0.0;` add:

```js
               initNoiseCache();
               buildWCACells();
```

- [ ] **Step 4: Rebuild cell list periodically + raise substep cap in `tick` handler**

Inside the `if (msg.type === 'tick')` block, find:
```js
                   const substeps = Math.max(1, Math.min(20, Math.round(targetSteps)));
```
Change `20` to `40`:
```js
                   const substeps = Math.max(1, Math.min(40, Math.round(targetSteps)));
```

After the substep loop `for (let i = 0; i < substeps; i++) integrate();`, add:
```js
                   if (stepCount % 8 === 0) buildWCACells();
```

- [ ] **Step 5: Verify in browser**

Load `2HHB`. Start simulation. Open DevTools Console — no errors. Load `8PTU` (large glycoprotein, ~5000 atoms, previously WCA was disabled). Simulation should run and produce repulsion-driven motion.

- [ ] **Step 6: Commit**

```bash
cd /home/dzyla/ProteinCHAOS
git add index.html
git commit -m "perf: cell-list WCA (O(N)), Gaussian noise cache, substep cap 40"
```

---

## Task 2: Rotation matrix precompute in `accumulateChaos`

**Files:**
- Modify: `index.html` — `accumulateChaos` function inside `workerCode`, lines ~991–1092

- [ ] **Step 1: Replace the per-link rotation code**

Find the section inside `accumulateChaos` that starts with:
```js
          // Rotation Math (Euler X -> Y -> Z)
          const radX = (params.rotX || 0) * 0.0174533;
          const radY = (params.rotY || 0) * 0.0174533;
          const radZ = (params.rotZ || 0) * 0.0174533;
          
          const cx = Math.cos(radX), sx = Math.sin(radX);
          const cy = Math.cos(radY), sy = Math.sin(radY);
          const cz = Math.cos(radZ), sz = Math.sin(radZ);
```
Replace it (and keep everything before the `for (let k = 0; k < list.length; k++)` loop) with:

```js
          // Precompute combined rotation matrix R = Rz·Ry·Rx (9 trig calls once vs per-bond)
          const radX = (params.rotX || 0) * 0.0174533;
          const radY = (params.rotY || 0) * 0.0174533;
          const radZ = (params.rotZ || 0) * 0.0174533;
          const cxr = Math.cos(radX), sxr = Math.sin(radX);
          const cyr = Math.cos(radY), syr = Math.sin(radY);
          const czr = Math.cos(radZ), szr = Math.sin(radZ);
          // R = Rz * Ry * Rx
          const m00 = cyr*czr,              m01 = czr*sxr*syr - cxr*szr, m02 = cxr*czr*syr + sxr*szr;
          const m10 = cyr*szr,              m11 = cxr*czr + sxr*syr*szr,  m12 = cxr*syr*szr - czr*sxr;
          const m20 = -syr,                 m21 = cyr*sxr,                m22 = cxr*cyr;
```

- [ ] **Step 2: Replace per-link rotation application**

Inside the `for (let k = 0; k < list.length; k++)` loop, find the two rotation blocks labeled "Apply Rotation P1" and "Apply Rotation P2" and replace both with matrix-vector multiply:

```js
              // Rotate P1
              const rx1 = m00*x1 + m01*y1 + m02*z1;
              const ry1 = m10*x1 + m11*y1 + m12*z1;
              const rz1 = m20*x1 + m21*y1 + m22*z1;
              x1 = rx1; y1 = ry1; z1 = rz1;

              // Rotate P2
              const rx2 = m00*x2 + m01*y2 + m02*z2;
              const ry2 = m10*x2 + m11*y2 + m12*z2;
              const rz2 = m20*x2 + m21*y2 + m22*z2;
              x2 = rx2; y2 = ry2; z2 = rz2;
```

Also update the depth cue factor from `0.05` to `0.12` for stronger depth perception:
```js
                      const depthCue = Math.max(0.05, 1.0 + z * 0.12);
```

- [ ] **Step 3: Verify**

Load `2HHB`, run simulation. Protein should rotate correctly with camera sliders. No visual change expected — just confirm correct behavior and no console errors.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "perf: precompute rotation matrix once per frame in accumulateChaos"
```

---

## Task 3: Ping-pong (pre-allocated) transferable send buffer

**Files:**
- Modify: `index.html` — worker `init` + `tick` handlers (~lines 1096–1189); main `handleWorkerMessage` (~line 1746)

The goal: eliminate `new Float32Array(accumulationBuffer)` each frame (heap allocation + GC pressure). Instead, `sentBuffer` is pre-allocated once and bounced as a transferable.

- [ ] **Step 1: Update worker `init` handler to pre-allocate `sentBuffer`**

Inside the `if (msg.type === 'init')` block, find:
```js
               sentPositions = new Float32Array(nAtoms * 3);
               sentBuffer = null;
```
Replace with:
```js
               sentPositions = new Float32Array(nAtoms * 3);
               const sCh = (colorMode === 'chain') ? 3 : 1;
               sentBuffer = new Float32Array(artWidth * artHeight * sCh);
```

- [ ] **Step 2: Update worker `resize` + `params` (colorMode change) handlers**

Inside `if (msg.type === 'resize')`:
```js
               artWidth = msg.width; artHeight = msg.height;
               const channels = (colorMode === 'chain') ? 3 : 1;
               accumulationBuffer = new Float32Array(artWidth * artHeight * channels);
               sentBuffer = new Float32Array(artWidth * artHeight * channels);
               fadeCursor = 0; needsFrame = true;
```

Inside `if (msg.type === 'params')`, after the colorMode branch that rebuilds `accumulationBuffer`, add:
```js
                   sentBuffer = new Float32Array(artWidth * artHeight * channels);
```

- [ ] **Step 3: Update the tick send section to use transfer**

Find in the `tick` handler:
```js
              let bufToSend = null;
              if (shouldSendImage && accumulationBuffer && nAtoms > 0) bufToSend = new Float32Array(accumulationBuffer);
              if (bufToSend) postMessage({ type: 'update', positions: sentPositions, buffer: bufToSend, time: simTime }, [bufToSend.buffer]);
              else postMessage({ type: 'update', positions: sentPositions, time: simTime });
```
Replace with:
```js
              if (shouldSendImage && accumulationBuffer && nAtoms > 0 && sentBuffer !== null) {
                  sentBuffer.set(accumulationBuffer);
                  const toSend = sentBuffer;
                  sentBuffer = null; // in-flight until main returns it
                  postMessage({ type: 'update', positions: sentPositions, buffer: toSend, time: simTime }, [toSend.buffer]);
              } else {
                  postMessage({ type: 'update', positions: sentPositions, time: simTime });
              }
```

- [ ] **Step 4: Add `buf_return` handler in worker `onmessage`**

Inside `onmessage`, add after the last `else if` before the closing `}`:
```js
           } else if (msg.type === 'buf_return') {
               // Main thread returned the buffer after rendering — re-adopt it
               sentBuffer = new Float32Array(msg.buffer);
```

- [ ] **Step 5: Return buffer in main thread after `renderDisplay`**

In `handleWorkerMessage`, find:
```js
              if (buf) {
                  const isHistoryMode = !ui.history.disabled && (parseInt(ui.history.value) < parseInt(ui.history.max));
                  if (!isHistoryMode) { latestBuffer = buf; renderDisplay(buf); }
```
Replace with:
```js
              if (buf) {
                  const isHistoryMode = !ui.history.disabled && (parseInt(ui.history.value) < parseInt(ui.history.max));
                  if (!isHistoryMode) { latestBuffer = new Float32Array(buf); renderDisplay(buf); }
                  // Return buffer to worker (ping-pong)
                  if (worker) worker.postMessage({ type: 'buf_return', buffer: buf }, [buf.buffer]);
```

Note: `latestBuffer = new Float32Array(buf)` makes a copy for history/snapshot purposes BEFORE we return the buffer to the worker.

- [ ] **Step 6: Verify**

Load `2HHB`, run 30 seconds. No console errors. DevTools Memory tab: allocation rate should be significantly reduced (no large Float32Array allocs per frame in fast mode). Snapshot and history functions should still work correctly.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "perf: ping-pong transferable send buffer, eliminate per-frame Float32Array alloc"
```

---

## Task 4: ACES filmic tone mapping

**Files:**
- Modify: `index.html` — `renderDisplay` function, ~lines 1794–1881

ACES produces richer shadows and graceful highlight roll-off compared to the current log-compression.

- [ ] **Step 1: Add `acesToneMap` function before `renderDisplay`**

Insert just before `function renderDisplay(buffer) {`:

```js
      function acesToneMap(x) {
          // ACES fitted curve (Hill 2015)
          if (x <= 0) return 0;
          const v = (x*(2.51*x+0.03)) / (x*(2.43*x+0.59)+0.14);
          return v < 0 ? 0 : v > 1 ? 1 : v;
      }
```

- [ ] **Step 2: Replace tone mapping in density path of `renderDisplay`**

Find inside `renderDisplay`, the density path (inside `if (!isRGB)`):
```js
                  if (val > 0.0) {
                      let norm = Math.log(val + 1) / logDen;
                      if (norm < 0) norm = 0; if (norm > 1) norm = 1;
                      norm = Math.pow(norm, invGamma);
```
Replace with:
```js
                  if (val > 0.0) {
                      let norm = acesToneMap(val / displayMax);
                      norm = Math.pow(norm, invGamma);
```

- [ ] **Step 3: Replace tone mapping in RGB chain path**

Find the RGB path (inside the `else` block of `if (!isRGB)`):
```js
                  // Apply Log Compression (Auto-Exposure)
                  rVal = Math.log(rVal + 1) / logDen;
                  gVal = Math.log(gVal + 1) / logDen;
                  bVal = Math.log(bVal + 1) / logDen;
                  
                  // Clamp 0-1
                  if (rVal > 1) rVal = 1; if (rVal < 0) rVal = 0;
                  if (gVal > 1) gVal = 1; if (gVal < 0) gVal = 0;
                  if (bVal > 1) bVal = 1; if (bVal < 0) bVal = 0;

                  // Apply Gamma
                  rVal = Math.pow(rVal, invGamma);
                  gVal = Math.pow(gVal, invGamma);
                  bVal = Math.pow(bVal, invGamma);
```
Replace with:
```js
                  // ACES tone mapping + gamma
                  rVal = Math.pow(acesToneMap(rVal / displayMax), invGamma);
                  gVal = Math.pow(acesToneMap(gVal / displayMax), invGamma);
                  bVal = Math.pow(acesToneMap(bVal / displayMax), invGamma);
```

- [ ] **Step 4: Verify**

Load `2HHB`, start simulation 10s. In `fire` theme: highlights should not blow out to pure white. In `ink` theme: dark regions should have richer tone. Compare: `chaos-fire` preset looks dramatically more filmic — bright trails fade smoothly rather than clipping.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: ACES filmic tone mapping replaces log-compression in renderDisplay"
```

---

## Task 5: 2D canvas bloom pass

**Files:**
- Modify: `index.html` — `setupCanvasResolution` (~line 1959), `renderDisplay` (~line 1794), Style section HTML (~line 524), `bindUI` (~line 2151), `ui` object (~line 1264)

- [ ] **Step 1: Add bloom canvas state variables**

Near the top of the main-thread variable declarations (around line 1245, after `let grainMap = null;`), add:

```js
      let bloomCanvas = null;
      let bloomCtx = null;
```

- [ ] **Step 2: Initialize bloom canvas in `setupCanvasResolution`**

At the end of `setupCanvasResolution()`, after the grain map generation block, add:

```js
          bloomCanvas = document.createElement('canvas');
          bloomCanvas.width  = Math.max(1, artWidth  >> 2);
          bloomCanvas.height = Math.max(1, artHeight >> 2);
          bloomCtx = bloomCanvas.getContext('2d');
```

- [ ] **Step 3: Apply bloom in `renderDisplay` after `putImageData`**

Find in `renderDisplay`:
```js
          artCtx.putImageData(viewImageData, 0, 0);
          if (state.isRecording) {
```
Replace with:
```js
          artCtx.putImageData(viewImageData, 0, 0);

          // Bloom pass (screen-blend downsampled blur)
          const bloomVal = parseFloat(ui.bloom ? ui.bloom.value : 0);
          if (bloomVal > 0.005 && bloomCanvas) {
              const bpx = Math.max(1, Math.round(bloomCanvas.width / 64));
              bloomCtx.filter = `blur(${bpx}px)`;
              bloomCtx.drawImage(artCanvas, 0, 0, bloomCanvas.width, bloomCanvas.height);
              bloomCtx.filter = 'none';
              artCtx.save();
              artCtx.globalCompositeOperation = 'screen';
              artCtx.globalAlpha = bloomVal;
              artCtx.drawImage(bloomCanvas, 0, 0, artWidth, artHeight);
              artCtx.restore();
          }

          if (state.isRecording) {
```

- [ ] **Step 4: Add Bloom slider to Style section HTML**

In the HTML Style Settings section (after the Jitter slider, before the closing `</div>` of `<!-- 4. Visuals -->`), add:

```html
          <div class="slider-container" title="Glow / bloom intensity on bright trails.">
            <div class="slider-header"><span>Bloom / Glow</span><span id="bloom-val" class="slider-val">0.15</span></div>
            <input type="range" id="art-bloom" min="0" max="0.5" value="0.15" step="0.01" />
          </div>
```

- [ ] **Step 5: Wire bloom slider in `ui` object and `bindUI`**

In the `ui` object definition, after `jitterVal`:
```js
          bloom: document.getElementById('art-bloom'), bloomVal: document.getElementById('bloom-val'),
```

In `bindUI`, after the jitter binding:
```js
          if (ui.bloom) ui.bloom.oninput = (e) => { if(ui.bloomVal) ui.bloomVal.textContent = parseFloat(e.target.value).toFixed(2); renderDisplay(); };
```

- [ ] **Step 6: Verify**

Load `2HHB`, set theme to `fire`, bloom at 0.15. Hot bright pixels should have a visible glow halo. Set bloom to 0: no glow. Set to 0.4: dramatic neon glow. No GC spikes (bloomCanvas reused each frame).

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: 2D screen-blend bloom pass with bloom slider"
```

---

## Task 6: Tiled value-noise film grain

**Files:**
- Modify: `index.html` — `setupCanvasResolution` (~line 1959), `renderDisplay` grain application (~line 1826)

Replaces the per-pixel sin-hash grain (32MB array at 4K) with a 256×256 tiled value-noise tile (256KB). Brightness-adaptive: bright areas get near-zero grain, dark areas get full grain.

- [ ] **Step 1: Replace grain generation in `setupCanvasResolution`**

Find:
```js
          grainMap = new Float32Array(artWidth * artHeight);
          for (let y = 0; y < artHeight; y++) { for (let x = 0; x < artWidth; x++) { const idx = y * artWidth + x; const v = Math.sin(x * 12.9898 + y * 78.233); grainMap[idx] = 0.9 + ((v - Math.floor(v)) * 0.1); } }
```
Replace with:
```js
          // 256×256 tiled value-noise grain (uses 256KB vs 32MB at 4K)
          const GTILE = 256, GGRID = 16;
          const gGrid = new Float32Array(GGRID * GGRID);
          for (let i = 0; i < gGrid.length; i++) gGrid[i] = Math.random();
          grainMap = new Float32Array(GTILE * GTILE);
          for (let gy = 0; gy < GTILE; gy++) {
              for (let gx = 0; gx < GTILE; gx++) {
                  const wx = gx * GGRID / GTILE, wy = gy * GGRID / GTILE;
                  const xi = Math.floor(wx) % GGRID, yi = Math.floor(wy) % GGRID;
                  const xi1 = (xi+1)%GGRID, yi1 = (yi+1)%GGRID;
                  const fx = wx - Math.floor(wx), fy = wy - Math.floor(wy);
                  const ux = fx*fx*(3-2*fx), uy = fy*fy*(3-2*fy);
                  const v = gGrid[yi*GGRID+xi]*(1-ux)*(1-uy)
                          + gGrid[yi*GGRID+xi1]*ux*(1-uy)
                          + gGrid[yi1*GGRID+xi]*(1-ux)*uy
                          + gGrid[yi1*GGRID+xi1]*ux*uy;
                  grainMap[gy*GTILE+gx] = v; // 0..1
              }
          }
```

- [ ] **Step 2: Replace grain application in `renderDisplay` density path**

Find:
```js
                  if (useGrain) { const grain = grainMap[i]; r *= grain; g *= grain; b *= grain; }
```
Replace with:
```js
                  if (useGrain && grainMap) {
                      const gx2 = (i % artWidth) & 255;
                      const gy2 = Math.floor(i / artWidth) & 255;
                      const rawG = grainMap[gy2*256+gx2]; // 0..1
                      const lum = (r + g + b) / 765.0;
                      const gs = 1.0 - lum * 0.75; // full grain in dark, 25% in white
                      const grain = 0.88 + rawG * 0.12 * gs;
                      r = Math.min(255, r * grain);
                      g = Math.min(255, g * grain);
                      b = Math.min(255, b * grain);
                  }
```

- [ ] **Step 3: Verify**

Load `2HHB`, set theme to `rough`. Grain should look organic — no diagonal banding. In `ink` theme (white background), white areas should be grain-free; dark trails have visible grain. Memory footprint reduced: grainMap is 256KB instead of up to 32MB.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: tiled value-noise film grain, brightness-adaptive, 256KB vs 32MB"
```

---

## Task 7: Auto-spin (slow tumble mode)

**Files:**
- Modify: `index.html` — `tick` function (~line 1314), Camera & Projection HTML section (~line 479), `ui` object (~line 1264), `bindUI` (~line 2151), main-thread state vars (~line 1235)

- [ ] **Step 1: Add spin state variables**

After `let currentPdbId = "1CDW";` (around line 1254), add:

```js
      let autoSpinSpeed = 0;   // degrees/second, 0 = off
      let autoSpinAngle = 0;
      let autoSpinLastTime = 0;
```

- [ ] **Step 2: Add spin logic to `tick`**

In the `tick(now)` function, just before `if (worker && state.hasStructure && !workerTickInFlight)`, add:

```js
          if (autoSpinSpeed !== 0 && state.hasStructure) {
              if (autoSpinLastTime === 0) autoSpinLastTime = now;
              const spinDt = Math.min(now - autoSpinLastTime, 100); // cap at 100ms
              autoSpinLastTime = now;
              autoSpinAngle = (autoSpinAngle + autoSpinSpeed * spinDt / 1000 + 360) % 360;
              ui.rotY.value = autoSpinAngle.toFixed(1);
              window.updateWorkerParams();
          } else {
              autoSpinLastTime = 0;
          }
```

- [ ] **Step 3: Add spin UI elements to Camera section HTML**

At the end of the Camera & Projection section (just before its closing `</div>`), add:

```html
             <div class="slider-container mt-2">
               <div class="slider-header">
                 <span>Auto-Spin</span>
                 <label class="flex items-center gap-1 cursor-pointer">
                   <input type="checkbox" id="spin-toggle" class="w-3 h-3 accent-blue-500" onchange="toggleAutoSpin(this.checked)" />
                   <span id="spin-val" class="slider-val">0 °/s</span>
                 </label>
               </div>
               <input type="range" id="spin-speed" min="-60" max="60" value="5" step="1" oninput="updateSpinSpeed(this.value)" />
             </div>
```

- [ ] **Step 4: Add spin control functions and wire `ui`**

In the `ui` object, add:
```js
          spinToggle: document.getElementById('spin-toggle'),
          spinSpeed: document.getElementById('spin-speed'),
          spinVal: document.getElementById('spin-val'),
```

After `bindUI()` (or at the end of the main script), add:

```js
      window.toggleAutoSpin = (on) => {
          if (on) {
              autoSpinSpeed = parseFloat(ui.spinSpeed ? ui.spinSpeed.value : 5);
              autoSpinAngle = parseFloat(ui.rotY.value) || 0;
          } else {
              autoSpinSpeed = 0;
              autoSpinLastTime = 0;
          }
      };
      window.updateSpinSpeed = (val) => {
          if (ui.spinVal) ui.spinVal.textContent = val + ' °/s';
          if (autoSpinSpeed !== 0) autoSpinSpeed = parseFloat(val);
      };
```

- [ ] **Step 5: Verify**

Load `2HHB`, start simulation, enable Auto-Spin at 5°/s. Protein should slowly rotate, accumulating beautiful spiral trails. Change speed to -10°/s: reverses. Uncheck toggle: stops spinning, Yaw slider frozen at current angle.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: auto-spin (slow tumble) mode with speed slider in Camera section"
```

---

## Task 8: New chain palettes + residue-property color constants

**Files:**
- Modify: `index.html` — `CHAIN_PALETTES` (~line 1200), `PRESETS` section, theme dropdown HTML (~line 555)

- [ ] **Step 1: Add new palettes to `CHAIN_PALETTES`**

Find the end of the `CHAIN_PALETTES` object (after `'earth': [...]`), add before the closing `};`:

```js
          'vivid':       [new THREE.Color(0xff3b5c), new THREE.Color(0xff9f0a), new THREE.Color(0x30d158), new THREE.Color(0x64d2ff), new THREE.Color(0xbf5af2), new THREE.Color(0xff6961), new THREE.Color(0xffd60a), new THREE.Color(0x0a84ff)],
          'publication': [new THREE.Color(0x222222), new THREE.Color(0x555555), new THREE.Color(0x888888), new THREE.Color(0xbbbbbb), new THREE.Color(0x1a6b8a), new THREE.Color(0x8a3324), new THREE.Color(0x4a7c59), new THREE.Color(0x7a4f2e)],
          'wong':        [new THREE.Color(0xe69f00), new THREE.Color(0x56b4e9), new THREE.Color(0x009e73), new THREE.Color(0xf0e442), new THREE.Color(0x0072b2), new THREE.Color(0xd55e00), new THREE.Color(0xcc79a7), new THREE.Color(0x000000)],
          'pastel':      [new THREE.Color(0xffb3ba), new THREE.Color(0xffdfba), new THREE.Color(0xffffba), new THREE.Color(0xbaffc9), new THREE.Color(0xbae1ff), new THREE.Color(0xd4baff), new THREE.Color(0xffbaee), new THREE.Color(0xc9f5f5)],
```

- [ ] **Step 2: Add `RESIDUE_PROPERTIES` table**

After `const GLYCAN_NAMES = new Set([...]);` (line ~1222), add:

```js
      // Residue property colors [r, g, b] normalized 0-1
      const RESIDUE_PROPERTIES = {
          // Hydrophobic (orange)
          ALA:[0.98,0.45,0.09], VAL:[0.98,0.45,0.09], ILE:[0.98,0.45,0.09],
          LEU:[0.98,0.45,0.09], MET:[0.98,0.45,0.09], PHE:[0.98,0.45,0.09],
          TRP:[0.98,0.45,0.09], PRO:[0.98,0.45,0.09],
          // Polar uncharged (cyan)
          SER:[0.13,0.83,0.94], THR:[0.13,0.83,0.94], CYS:[0.13,0.83,0.94],
          TYR:[0.13,0.83,0.94], ASN:[0.13,0.83,0.94], GLN:[0.13,0.83,0.94],
          // Positive (blue)
          LYS:[0.23,0.51,0.96], ARG:[0.23,0.51,0.96], HIS:[0.43,0.55,0.96],
          // Negative (red)
          ASP:[0.94,0.27,0.27], GLU:[0.94,0.27,0.27],
          // Glycine/special (light gray)
          GLY:[0.82,0.82,0.82],
          // Default
          UNK:[0.55,0.55,0.55]
      };
```

- [ ] **Step 3: Add new chain palette options to theme dropdown**

In the theme `<select id="art-theme">`, inside the `<optgroup label="Chain Colors">`, add after `<option value="chain-earth">Chain: Earth</option>`:

```html
                  <option value="chain-vivid">Chain: Vivid</option>
                  <option value="chain-publication">Chain: Publication</option>
                  <option value="chain-wong">Chain: Wong (Colorblind-safe)</option>
                  <option value="chain-pastel">Chain: Pastel</option>
```

- [ ] **Step 4: Verify**

Load `2HHB`. Switch theme to `Chain: Vivid` — four chains should appear in red, orange, green, cyan. Switch to `Chain: Wong` — colorblind-safe palette. No console errors.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: vivid/publication/wong/pastel palettes + RESIDUE_PROPERTIES table"
```

---

## Task 9: Per-chain color pickers + color mode UI + residue coloring

**Files:**
- Modify: `index.html` — HTML sidebar (~line 622), `parseAndLoad` (~line 1363), main-thread state vars, `updateWorkerParams`, `update3DColors`, `updateChainColors`, `bindUI`

- [ ] **Step 1: Add chain color section HTML placeholder**

In the sidebar HTML, after the `<!-- 4. Visuals -->` section's closing `</div>` and before `<!-- 5. Data Actions -->`, insert:

```html
        <!-- Chain Colors (populated dynamically on load) -->
        <div id="chain-color-section" class="control-section" style="display:none"></div>
```

- [ ] **Step 2: Add `art-color-mode` dropdown to Style section HTML**

Inside the Style section, after the Theme `<select>` block's `</div>`, add:

```html
          <div class="mb-4">
            <label class="text-xs text-gray-500 block mb-1">Color Mode</label>
            <select id="art-color-mode" class="w-full" onchange="onColorModeChange()">
              <option value="density" selected>Density (Theme)</option>
              <option value="chain">Chain Colors</option>
              <option value="residue">Residue Properties</option>
            </select>
          </div>
```

- [ ] **Step 3: Add `window.chainColors` state and `rebuildColorArray` function**

After `let currentPdbId = "1CDW";`, add:
```js
      window.chainColors = {}; // chain letter → hex string e.g. '#93c5fd'
```

Before `function bindUI()`, add the following two functions:

```js
      function rebuildColorArray() {
          if (!window.globalAtoms) return;
          const cMode = document.getElementById('art-color-mode')?.value || 'chain';
          const colorArray = new Float32Array(window.globalAtoms.length * 3);
          for (let i = 0; i < window.globalAtoms.length; i++) {
              const atom = window.globalAtoms[i];
              let r, g, b;
              if (cMode === 'residue') {
                  if (atom.isNucleic) { r=0.68; g=0.54; b=0.99; }
                  else if (atom.isGlycan) { r=0.97; g=0.62; b=0.07; }
                  else {
                      const p = RESIDUE_PROPERTIES[atom.resName] || RESIDUE_PROPERTIES['UNK'];
                      r=p[0]; g=p[1]; b=p[2];
                  }
              } else {
                  const hex = window.chainColors[atom.chain] || '#93c5fd';
                  r = parseInt(hex.slice(1,3),16)/255;
                  g = parseInt(hex.slice(3,5),16)/255;
                  b = parseInt(hex.slice(5,7),16)/255;
              }
              colorArray[i*3]=r; colorArray[i*3+1]=g; colorArray[i*3+2]=b;
          }
          if (worker) worker.postMessage({ type: 'colors', data: colorArray });
          window.update3DColors();
      }

      window.updateChainColor = (chain, hex) => {
          window.chainColors[chain] = hex;
          rebuildColorArray();
      };

      window.onColorModeChange = () => {
          window.updateWorkerParams(); // switches density vs chain buffer mode
          rebuildColorArray();
          window.clearArt(false);
      };
```

- [ ] **Step 4: Build chain color picker section dynamically in `parseAndLoad`**

Inside `parseAndLoad`, find the "5. Restraints & Colors" section comment and the existing palette initialization block:
```js
                  // Use DEFAULT palette for initialization
                  const palette = CHAIN_PALETTES['default'];
                  const chainMap = {}; let chainCount = 0;
                  const colorArray = new Float32Array(newAtoms.length * 3);
                  
                  for(let i=0; i<newAtoms.length; i++) {
                      const atom = newAtoms[i];
                      if(chainMap[atom.chain] === undefined) chainMap[atom.chain] = chainCount++;
                      const color = palette[chainMap[atom.chain] % palette.length];
                      colorArray[i*3] = color.r;
                      colorArray[i*3+1] = color.g;
                      colorArray[i*3+2] = color.b;
                  }
```
Replace with:
```js
                  // Build per-chain color state and picker UI
                  const palette = CHAIN_PALETTES['default'];
                  const uniqueChains = [...new Set(newAtoms.map(a => a.chain))].sort();
                  window.chainColors = {};
                  uniqueChains.forEach((ch, idx) => {
                      const c = palette[idx % palette.length];
                      const hex = '#'
                          + Math.round(c.r*255).toString(16).padStart(2,'0')
                          + Math.round(c.g*255).toString(16).padStart(2,'0')
                          + Math.round(c.b*255).toString(16).padStart(2,'0');
                      window.chainColors[ch] = hex;
                  });

                  const chainSection = document.getElementById('chain-color-section');
                  if (chainSection) {
                      const chainCounts = {};
                      newAtoms.forEach(a => { chainCounts[a.chain] = (chainCounts[a.chain]||0)+1; });
                      let html = '<div class="section-title" style="margin-bottom:8px">Chains</div>';
                      uniqueChains.forEach(ch => {
                          const hex = window.chainColors[ch];
                          html += `<div class="flex items-center gap-2 mb-1">
                              <input type="color" value="${hex}" oninput="updateChainColor('${ch}', this.value)"
                                     style="width:24px;height:24px;border:none;background:none;padding:0;cursor:pointer;border-radius:4px;">
                              <span class="text-xs text-gray-300 font-mono">Chain ${ch}</span>
                              <span class="text-[10px] text-gray-500 ml-auto">${chainCounts[ch]} res</span>
                          </div>`;
                      });
                      chainSection.innerHTML = html;
                      chainSection.style.display = uniqueChains.length > 0 ? 'block' : 'none';
                  }

                  // Build initial color array from chain colors
                  const colorArray = new Float32Array(newAtoms.length * 3);
                  for(let i=0; i<newAtoms.length; i++) {
                      const atom = newAtoms[i];
                      const hex = window.chainColors[atom.chain] || '#93c5fd';
                      colorArray[i*3]   = parseInt(hex.slice(1,3),16)/255;
                      colorArray[i*3+1] = parseInt(hex.slice(3,5),16)/255;
                      colorArray[i*3+2] = parseInt(hex.slice(5,7),16)/255;
                  }
```

- [ ] **Step 5: Update `updateWorkerParams` to read from `art-color-mode` dropdown**

Find in `updateWorkerParams`:
```js
          let cmode = 'density';
          if (ui.theme.value.startsWith('chain')) cmode = 'chain';
```
Replace with:
```js
          let cmode = 'density';
          const artCM = document.getElementById('art-color-mode')?.value || 'density';
          if (artCM === 'chain' || artCM === 'residue') cmode = 'chain';
```

- [ ] **Step 6: Update `update3DColors` to use `window.chainColors`**

Find the chain mode branch inside `update3DColors`:
```js
            if (mode === 'chain') {
               if (chainMap[atom.chain] === undefined) chainMap[atom.chain] = chainCount++;
               color = currentChainPalette[chainMap[atom.chain] % currentChainPalette.length].clone();
```
Replace with:
```js
            if (mode === 'chain') {
               const hex = window.chainColors && window.chainColors[atom.chain]
                   ? window.chainColors[atom.chain] : '#93c5fd';
               color = new THREE.Color(hex);
```

- [ ] **Step 7: Update `updateChainColors` (the old palette-based one) to call `rebuildColorArray`**

The old `updateChainColors` function (lines ~2124–2149) is now superseded. Replace its body with:

```js
      function updateChainColors() {
          // Legacy: called when chain-* theme selected. Now just triggers rebuildColorArray.
          rebuildColorArray();
      }
```

- [ ] **Step 8: Wire `art-color-mode` in `bindUI`**

In `bindUI`, the `ui.theme.onchange` already calls `updateChainColors()` for chain themes. Also add the new dropdown. Find `ui.source.onchange`, and before it add:

```js
          // Color mode change is handled by onColorModeChange() directly via HTML onchange
```

(No code needed — it's wired via the `onchange="onColorModeChange()"` attribute added in Step 2.)

- [ ] **Step 9: Verify**

Load `2HHB` (4 chains A, B, C, D). Chain color section appears with 4 color swatches. Click a swatch and change color — art canvas and 3D view update immediately. Switch Color Mode to "Residue Properties" — helices/sheets colored by property. Switch to "Density (Theme)" — returns to single-color theme rendering.

- [ ] **Step 10: Commit**

```bash
git add index.html
git commit -m "feat: per-chain color pickers, residue-property coloring, color mode decoupled from theme"
```

---

## Task 10: UX fixes — clipboard copy + presets cleanup + title update

**Files:**
- Modify: `index.html` — Data Actions HTML (~line 622), `downloadHighRes`/`saveSnapshot` area, title

- [ ] **Step 1: Add clipboard copy button to Data Actions section**

Find the button row:
```html
          <div class="flex gap-2 mb-3">
            <button onclick="clearArt(false)" class="btn btn-danger text-xs">Clear Canvas</button>
            <button onclick="saveSnapshot()" class="btn btn-secondary text-xs">Snapshot</button>
          </div>
```
Replace with:
```html
          <div class="flex gap-2 mb-3">
            <button onclick="clearArt(false)" class="btn btn-danger text-xs">Clear</button>
            <button onclick="saveSnapshot()" class="btn btn-secondary text-xs">Snapshot</button>
            <button onclick="copyToClipboard()" class="btn btn-secondary text-xs" title="Copy current canvas to clipboard">Copy</button>
          </div>
```

- [ ] **Step 2: Add `copyToClipboard` function**

After `window.downloadHighRes`, add:

```js
      window.copyToClipboard = () => {
          if (!latestBuffer) { alert("Nothing to copy yet — run the simulation first."); return; }
          artCanvas.toBlob(blob => {
              if (!blob) return;
              try {
                  navigator.clipboard.write([new ClipboardItem({'image/png': blob})])
                      .then(() => { /* silent success */ })
                      .catch(() => alert("Clipboard write failed — try downloading instead."));
              } catch(e) { alert("Clipboard API not available in this browser."); }
          });
      };
```

- [ ] **Step 3: Update page title**

Find `<title>ProteinCHAOS v1.1</title>` and change to `<title>ProteinCHAOS v1.2</title>`.

- [ ] **Step 4: Verify**

Load `2HHB`, run 5 seconds. Click Copy — paste into image editor or Slack, should get the current canvas as PNG. On browsers without Clipboard API (e.g., non-HTTPS), the fallback alert appears.

- [ ] **Step 5: Final verify — full regression**

Run the full app checklist:
- Load `2HHB` via Fetch: structure loads, 4 chains, chain color pickers appear
- Start simulation: physics running, trails accumulate
- Auto-spin: enable, protein tumbles, trails spiral
- Bloom: set to 0.3, bright trails glow
- Grain: set theme to `rough`, grain visible in dark areas only
- Color modes: switch Density/Chain/Residue, all update correctly
- Download HD: downloads `chaos_vc_2HHB_<theme>.png`
- Copy: copies to clipboard
- Record: 5s video downloads
- Load large structure `8PTU`: WCA cell-list active (no freeze), repulsion visible

- [ ] **Step 6: Final commit**

```bash
git add index.html
git commit -m "feat: clipboard copy, v1.2 title, UX cleanup — ProteinCHAOS v1.2 complete"
```

---

## Self-Review

**Spec coverage check:**
1. Cell-list WCA ✅ Task 1
2. Rotation matrix precompute ✅ Task 2
3. Noise cache ✅ Task 1
4. Ping-pong transferable buffers ✅ Task 3
5. ACES tone mapping ✅ Task 4
6. Depth cueing enhanced ✅ Task 2 (0.05→0.12 in accumulateChaos)
7. Bloom pass ✅ Task 5
8. Auto-spin ✅ Task 7
9. Film grain ✅ Task 6
10. Per-chain color pickers ✅ Task 9
11. New palettes (vivid/publication/wong/pastel) ✅ Task 8
12. Residue-property coloring ✅ Task 9
13. UX fixes (clipboard, color mode decoupled, title) ✅ Task 10

**Placeholder scan:** None found — all steps contain complete code.

**Type consistency check:**
- `window.chainColors` defined in Task 9 Step 3, used in Task 9 Steps 4/6 ✅
- `rebuildColorArray()` defined in Task 9 Step 3, called in Steps 5/6/7 ✅
- `bloomCanvas`/`bloomCtx` defined in Task 5 Step 1, used in Steps 2/3 ✅
- `autoSpinSpeed`/`autoSpinAngle`/`autoSpinLastTime` defined in Task 7 Step 1, used in Step 2 ✅
- `acesToneMap()` defined in Task 4 Step 1, used in Steps 2/3 ✅
- `RESIDUE_PROPERTIES` defined in Task 8 Step 2, used in Task 9 Step 3 ✅
- `buildWCACells()` / `initNoiseCache()` defined in Task 1 Step 1, called in Steps 3/4 ✅
- `noiseCache` / `noiseCursor` / `wcaCells` declared in Task 1 Step 1, used throughout worker ✅
- `ui.bloom` / `ui.bloomVal` added in Task 5 Steps 4/5, referenced in Task 5 Step 3 ✅
- `ui.spinToggle` / `ui.spinSpeed` / `ui.spinVal` added in Task 7 Step 4 ✅
- `window.updateChainColor` defined Task 9 Step 3, used in dynamically-built HTML (Task 9 Step 4) ✅
- `window.onColorModeChange` defined Task 9 Step 3, used in HTML `onchange` (Task 9 Step 2) ✅
