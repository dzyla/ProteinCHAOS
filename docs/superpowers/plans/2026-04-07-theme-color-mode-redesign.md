# Theme & Color Mode Redesign — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Cleanly separate density / chain / residue color modes so no broken combinations are possible, fix the black-background rendering bug in chain mode, add chain style effects, and eliminate CPU waste when paused.

**Architecture:** Single file `index.html`. Color Mode becomes the master switch; sub-panels for each mode are shown/hidden in the DOM. The worker gets an early-return guard when paused so it no longer burns CPU at 60fps doing nothing.

**Tech Stack:** Vanilla JS + HTML, Web Worker (embedded as Blob string), no build tools.

---

## Task 1: CPU Fix — Worker Early Return When Paused

**Files:**
- Modify: `index.html` — `workerCode` string, the `msg.type === 'tick'` handler (~line 1250)

The current `shouldSendImage` includes `|| !isRunning`, so a full buffer transfer fires every frame when paused. Also, even the position copy loop runs on every tick regardless.

- [ ] **Step 1.1: Add early-return guard in worker tick handler**

Find this exact block (around line 1250 inside the `workerCode` template literal):
```js
           } else if (msg.type === 'tick') {
               if (nAtoms === 0) { postMessage({ type: 'idle' }); return; }
               const dtInput = msg.dt || 16.6; 
```
Replace with:
```js
           } else if (msg.type === 'tick') {
               if (nAtoms === 0) { postMessage({ type: 'idle' }); return; }
               if (!isRunning && !needsFrame) { postMessage({ type: 'idle' }); return; }
               const dtInput = msg.dt || 16.6; 
```

- [ ] **Step 1.2: Remove `!isRunning` from shouldSendImage**

Find (around line 1273):
```js
              const shouldSendImage = forceHighFPS || (frameCounter % displayFrameSkip === 0) || !isRunning || needsFrame;
```
Replace with:
```js
              const shouldSendImage = forceHighFPS || (frameCounter % displayFrameSkip === 0) || needsFrame;
```

- [ ] **Step 1.3: Verify in browser**

Open `http://localhost:8080`, load a structure, start simulation, let it run for a few seconds, then pause. Check browser Task Manager (Shift+Esc in Chrome) — CPU for the page tab should drop close to zero within 1-2 seconds of pausing.

- [ ] **Step 1.4: Commit**

```bash
git add index.html
git commit -m "fix: stop worker CPU burn when paused — early idle return + remove !isRunning from shouldSendImage"
```

---

## Task 2: HTML — Restructure Style Controls Section

**Files:**
- Modify: `index.html` — the "Style Settings" control-section (~lines 542–654)

Remove the old block from `<!-- STYLE PRESET DROPDOWN -->` through `<!-- Chain Colors (populated dynamically on load) -->` and replace with the new structure below.

- [ ] **Step 2.1: Replace the style-settings control-section HTML**

Find this entire block (from the style preset label through the chain-color-section div):
```html
          <!-- STYLE PRESET DROPDOWN -->
          <div class="mb-4">
            <label class="text-xs text-gray-500 block mb-1">Style Preset</label>
            <select id="style-preset" class="w-full" onchange="applyPreset(this.value)">
              <option value="custom" selected>Custom</option>
              <optgroup label="Scientific / Editorial">
                  <option value="editorial-clean">Journal Figure (Ink)</option>
                  <option value="editorial-dark">Cover Art (Viridis)</option>
                  <option value="blueprint">Blueprint Technical</option>
                  <option value="lab">Clinical Lab</option>
              </optgroup>
              <optgroup label="Emotional / Abstract">
                  <option value="chaos-fire">Inferno Collapse</option>
                  <option value="chaos-gold">Golden Hour</option>
                  <option value="sketch">Rough Notebook</option>
                  <option value="cyber">Neon Cyberpunk</option>
              </optgroup>
            </select>
          </div>

          <div class="mb-4">
            <label class="text-xs text-gray-500 block mb-1">Theme</label>
            <select id="art-theme" class="w-full">
              <option value="sunset" selected>Sunset (Purple/Gold)</option>
              <optgroup label="Chain Colors">
                  <option value="chain-default">Chain: Default</option>
                  <option value="chain-spectral">Chain: Spectral</option>
                  <option value="chain-neon">Chain: Neon</option>
                  <option value="chain-retro">Chain: Retro</option>
                  <option value="chain-earth">Chain: Earth</option>
                  <option value="chain-vivid">Chain: Vivid</option>
                  <option value="chain-publication">Chain: Publication</option>
                  <option value="chain-wong">Chain: Wong (Colorblind-safe)</option>
                  <option value="chain-pastel">Chain: Pastel</option>
              </optgroup>
              <optgroup label="Density / Heatmaps">
                  <option value="fire">Inferno (Black Background)</option>
                  <option value="ice">Glacier (Cyan/Blue)</option>
                  <option value="ink">Sumi-e (Ink/White)</option>
                  <option value="rough">Rough Sketch (Pencil/Paper)</option>
                  <option value="charcoal">Charcoal (Dark Sketch)</option>
                  <option value="graphite">Graphite (Clean Pencil)</option>
                  <option value="sakura">Sakura (Pink/White)</option>
                  <option value="lab">Laboratory (Clinical)</option>
                  <option value="editorial">Editorial (Red/BW)</option>
                  <option value="bio">Biohazard (Green)</option>
                  <option value="blueprint">Blueprint (Tech)</option>
                  <option value="heme">Heme Glow (Deep Red)</option>
                  <option value="electrostatic">Electrostatic Field</option>
                  <option value="cyber">Neon Cyberpunk</option>
                  <option value="gold">Golden Hour</option>
                  <option value="quantum">Quantum Foam</option>
                  <option value="viridis">Viridis (Scientific)</option>
                  <option value="magma">Magma (Scientific)</option>
              </optgroup>
            </select>
          </div>

          <div class="mb-4">
            <label class="text-xs text-gray-500 block mb-1">Color Mode</label>
            <select id="art-color-mode" class="w-full" onchange="onColorModeChange()">
              <option value="density" selected>Density (Theme)</option>
              <option value="chain">Chain Colors</option>
              <option value="residue">Residue Properties</option>
            </select>
          </div>
```

Replace with:
```html
          <!-- COLOR MODE (master switch) -->
          <div class="mb-4">
            <label class="text-xs text-gray-500 block mb-1">Color Mode</label>
            <select id="art-color-mode" class="w-full" onchange="onColorModeChange()">
              <option value="density" selected>Density (Theme)</option>
              <option value="chain">Chain Colors</option>
              <option value="residue">Residue Properties</option>
            </select>
          </div>

          <!-- DENSITY SUB-PANEL -->
          <div id="density-panel">
            <div class="mb-4">
              <label class="text-xs text-gray-500 block mb-1">Theme</label>
              <select id="art-theme" class="w-full">
                <option value="sunset" selected>Sunset (Purple/Gold)</option>
                <option value="fire">Inferno (Black bg)</option>
                <option value="ice">Glacier (Cyan/Blue)</option>
                <option value="ink">Sumi-e (Ink/White)</option>
                <option value="rough">Rough Sketch (Pencil/Paper)</option>
                <option value="charcoal">Charcoal (Dark Sketch)</option>
                <option value="graphite">Graphite (Clean Pencil)</option>
                <option value="sakura">Sakura (Pink/White)</option>
                <option value="lab">Laboratory (Clinical)</option>
                <option value="editorial">Editorial (Red/BW)</option>
                <option value="bio">Biohazard (Green)</option>
                <option value="blueprint">Blueprint (Tech)</option>
                <option value="heme">Heme Glow (Deep Red)</option>
                <option value="electrostatic">Electrostatic Field</option>
                <option value="cyber">Neon Cyberpunk</option>
                <option value="gold">Golden Hour</option>
                <option value="quantum">Quantum Foam</option>
                <option value="viridis">Viridis (Scientific)</option>
                <option value="magma">Magma (Scientific)</option>
              </select>
            </div>
            <div class="mb-4">
              <label class="text-xs text-gray-500 block mb-1">Style Preset</label>
              <select id="style-preset" class="w-full" onchange="applyPreset(this.value)">
                <option value="custom" selected>Custom</option>
                <optgroup label="Scientific / Editorial">
                    <option value="editorial-clean">Journal Figure (Ink)</option>
                    <option value="editorial-dark">Cover Art (Viridis)</option>
                    <option value="blueprint">Blueprint Technical</option>
                    <option value="lab">Clinical Lab</option>
                </optgroup>
                <optgroup label="Emotional / Abstract">
                    <option value="chaos-fire">Inferno Collapse</option>
                    <option value="chaos-gold">Golden Hour</option>
                    <option value="sketch">Rough Notebook</option>
                    <option value="cyber">Neon Cyberpunk</option>
                </optgroup>
              </select>
            </div>
          </div>

          <!-- CHAIN COLORS SUB-PANEL -->
          <div id="chain-panel" style="display:none">
            <div class="mb-4">
              <label class="text-xs text-gray-500 block mb-1">Chain Palette</label>
              <select id="chain-palette" class="w-full" onchange="onChainPaletteChange()">
                <option value="default">Default</option>
                <option value="spectral">Spectral</option>
                <option value="neon">Neon</option>
                <option value="retro">Retro</option>
                <option value="earth">Earth</option>
                <option value="vivid">Vivid</option>
                <option value="publication">Publication</option>
                <option value="wong">Wong (Colorblind-safe)</option>
                <option value="pastel">Pastel</option>
              </select>
            </div>
            <div class="mb-4">
              <label class="text-xs text-gray-500 block mb-1">Chain Style</label>
              <select id="chain-style" class="w-full" onchange="renderDisplay()">
                <option value="normal">Normal</option>
                <option value="neon">Neon Glow</option>
                <option value="inkwash">Ink Wash</option>
                <option value="faded">Faded</option>
              </select>
            </div>
            <div class="mb-4">
              <label class="text-xs text-gray-500 block mb-1">Background</label>
              <select id="art-bg-mode" class="w-full" onchange="onBgModeChange()">
                <option value="black" selected>Black</option>
                <option value="white">White</option>
                <option value="dark">Dark Grey</option>
              </select>
            </div>
          </div>

          <!-- RESIDUE PROPERTIES SUB-PANEL -->
          <div id="residue-panel" style="display:none">
            <div class="mb-4">
              <label class="text-xs text-gray-500 block mb-1">Background</label>
              <select id="residue-bg-mode" class="w-full" onchange="onBgModeChange()">
                <option value="black" selected>Black</option>
                <option value="white">White</option>
                <option value="dark">Dark Grey</option>
              </select>
            </div>
          </div>
```

Also find and remove:
```html
        <!-- Chain Colors (populated dynamically on load) -->
        <div id="chain-color-section" class="control-section" style="display:none"></div>
```
Replace with (same, no change needed — it stays hidden by default and is shown/hidden by JS):
```html
        <!-- Chain Colors (populated dynamically on load) -->
        <div id="chain-color-section" class="control-section" style="display:none"></div>
```
(No change to this div — leave it as-is.)

- [ ] **Step 2.2: Verify HTML renders correctly in browser**

Open the page. Verify: Color Mode selector is at the top. Density panel (Theme + Style Preset) is visible. No chain-* options in the Theme dropdown. Switching Color Mode to "Chain Colors" shows `#chain-panel` (will be wired up in Task 3).

- [ ] **Step 2.3: Commit**

```bash
git add index.html
git commit -m "feat: restructure style controls — color mode as master switch, density/chain/residue sub-panels"
```

---

## Task 3: JS — New Helper Functions + Update onColorModeChange

**Files:**
- Modify: `index.html` — JS section, functions around lines 2344–2452

- [ ] **Step 3.1: Add `getChainBackground()` function**

Find the existing `updateArtPanelBackground` function:
```js
      function updateArtPanelBackground() {
          if (!artPanelEl) return;
          const theme = ui.theme.value;
          const bg = getThemeBackground(theme);
          artPanelEl.style.background = `rgb(${bg.r}, ${bg.g}, ${bg.b})`;
      }
```
Replace with:
```js
      function getChainBackground() {
          const mode = document.getElementById('art-color-mode')?.value || 'density';
          const selId = (mode === 'residue') ? 'residue-bg-mode' : 'art-bg-mode';
          const val = document.getElementById(selId)?.value || 'black';
          if (val === 'white') return {r:255, g:255, b:255};
          if (val === 'dark')  return {r:18,  g:18,  b:18 };
          return {r:0, g:0, b:0};
      }
      function updateArtPanelBackground() {
          if (!artPanelEl) return;
          const mode = document.getElementById('art-color-mode')?.value || 'density';
          const bg = (mode === 'density') ? getThemeBackground(ui.theme.value) : getChainBackground();
          artPanelEl.style.background = `rgb(${bg.r}, ${bg.g}, ${bg.b})`;
      }
```

- [ ] **Step 3.2: Add `onChainPaletteChange()` function**

Add right after the `updateArtPanelBackground` function:
```js
      window.onChainPaletteChange = () => {
          if (!window.globalAtoms) return;
          const paletteKey = document.getElementById('chain-palette')?.value || 'default';
          const palette = CHAIN_PALETTES[paletteKey] || CHAIN_PALETTES['default'];
          const uniqueChains = [...new Set(window.globalAtoms.map(a => a.chain))].sort();
          uniqueChains.forEach((ch, idx) => {
              const c = palette[idx % palette.length];
              window.chainColors[ch] = '#'
                  + Math.round(c.r*255).toString(16).padStart(2,'0')
                  + Math.round(c.g*255).toString(16).padStart(2,'0')
                  + Math.round(c.b*255).toString(16).padStart(2,'0');
          });
          const chainSection = document.getElementById('chain-color-section');
          if (chainSection) {
              chainSection.querySelectorAll('input[type="color"]').forEach((input, idx) => {
                  const ch = uniqueChains[idx];
                  if (ch) input.value = window.chainColors[ch];
              });
          }
          rebuildColorArray();
          window.clearArt(false);
      };

      window.onBgModeChange = () => {
          updateArtPanelBackground();
          renderDisplay();
      };
```

- [ ] **Step 3.3: Update `onColorModeChange()` to show/hide panels**

Find:
```js
      window.onColorModeChange = () => {
          window.updateWorkerParams(); // switches density vs chain buffer mode
          rebuildColorArray();
          window.clearArt(false);
      };
```
Replace with:
```js
      window.onColorModeChange = () => {
          const mode = document.getElementById('art-color-mode')?.value || 'density';
          document.getElementById('density-panel').style.display  = (mode === 'density')  ? '' : 'none';
          document.getElementById('chain-panel').style.display    = (mode === 'chain')    ? '' : 'none';
          document.getElementById('residue-panel').style.display  = (mode === 'residue')  ? '' : 'none';
          const chainSection = document.getElementById('chain-color-section');
          if (chainSection) chainSection.style.display = (mode === 'chain') ? '' : 'none';
          updateArtPanelBackground();
          window.updateWorkerParams();
          rebuildColorArray();
          window.clearArt(false);
      };
```

- [ ] **Step 3.4: Verify in browser**

Switch Color Mode to "Chain Colors" — the Theme/Preset panel hides, the Chain Palette/Style/Background panel appears, and the chain-color-section (per-chain pickers) appears if a structure is loaded. Switch to "Residue Properties" — only the Background select appears. Switch back to "Density" — Theme and Preset reappear.

- [ ] **Step 3.5: Commit**

```bash
git add index.html
git commit -m "feat: wire color mode panels — show/hide density/chain/residue sub-panels, add getChainBackground + onChainPaletteChange"
```

---

## Task 4: JS — Fix renderDisplay RGB Branch (bg bug + chain styles)

**Files:**
- Modify: `index.html` — `renderDisplay()` function, the `isRGB` branch (~line 2065)

- [ ] **Step 4.1: Replace the isRGB render branch**

Find the `bg` computation near the top of `renderDisplay` (just after `const theme = ui.theme.value;`):
```js
          const theme = ui.theme.value;
          const data = viewImageData.data;
          const bg = getThemeBackground(theme);
```
Replace with:
```js
          const theme = ui.theme.value;
          const data = viewImageData.data;
          const artMode = document.getElementById('art-color-mode')?.value || 'density';
          const bg = (artMode === 'density') ? getThemeBackground(theme) : getChainBackground();
```

- [ ] **Step 4.2: Fix zero-pixel background and add chain styles in the RGB branch**

Find:
```js
          } else {
              const len = buf.length / 3;
              for (let i = 0; i < len; i++) {
                  const idx = i * 4;
                  const r = toneMap(Math.log1p(buf[i*3])   / logDen);
                  const g = toneMap(Math.log1p(buf[i*3+1]) / logDen);
                  const b = toneMap(Math.log1p(buf[i*3+2]) / logDen);
                  data[idx] = r * 255; data[idx+1] = g * 255; data[idx+2] = b * 255; data[idx+3] = 255;
              }
          }
```
Replace with:
```js
          } else {
              const len = buf.length / 3;
              const chainStyle = document.getElementById('chain-style')?.value || 'normal';
              const bgLight = bg.r > 128; // used by inkwash to invert
              for (let i = 0; i < len; i++) {
                  const idx = i * 4;
                  let r = toneMap(Math.log1p(buf[i*3])   / logDen);
                  let g = toneMap(Math.log1p(buf[i*3+1]) / logDen);
                  let b = toneMap(Math.log1p(buf[i*3+2]) / logDen);
                  if (r === 0 && g === 0 && b === 0) {
                      data[idx] = bg.r; data[idx+1] = bg.g; data[idx+2] = bg.b; data[idx+3] = 255;
                      continue;
                  }
                  if (chainStyle === 'neon') {
                      const avg = (r + g + b) / 3;
                      r = Math.min(1, avg + (r - avg) * 1.8);
                      g = Math.min(1, avg + (g - avg) * 1.8);
                      b = Math.min(1, avg + (b - avg) * 1.8);
                  } else if (chainStyle === 'inkwash') {
                      const lum = 0.299*r + 0.587*g + 0.114*b;
                      r = lum + (r - lum) * 0.25;
                      g = lum + (g - lum) * 0.25;
                      b = lum + (b - lum) * 0.25;
                      if (bgLight) { r = 1 - r; g = 1 - g; b = 1 - b; }
                  } else if (chainStyle === 'faded') {
                      r *= 0.6; g *= 0.6; b *= 0.6;
                  }
                  data[idx] = r * 255; data[idx+1] = g * 255; data[idx+2] = b * 255; data[idx+3] = 255;
              }
          }
```

- [ ] **Step 4.3: Verify in browser**

Load a structure. Switch Color Mode to "Chain Colors". The background should now be black (matching the `art-bg-mode` select default). Switch background to "White" — background should become white with colored chain traces. Switch Chain Style to "Neon Glow" — traces should appear more saturated/vivid. Switch to "Ink Wash" on white background — traces should appear as dark/greyscale strokes on white. Switch to "Faded" — traces should appear softer/translucent.

- [ ] **Step 4.4: Commit**

```bash
git add index.html
git commit -m "fix: apply background in chain/residue render branch, add chain styles (neon/inkwash/faded)"
```

---

## Task 5: JS — Cleanup theme.onchange + chain-section visibility on load

**Files:**
- Modify: `index.html` — `bindUI()` around line 2425, and structure-load callback around line 1689

- [ ] **Step 5.1: Simplify `ui.theme.onchange` — remove chain-* palette loading**

Find:
```js
          ui.theme.onchange = () => {
              updateArtPanelBackground();
              window.updateWorkerParams(); // Switch mode
              if (ui.theme.value.startsWith('chain') && window.globalAtoms) {
                  // Load this palette's colors into per-chain pickers
                  const paletteKey = ui.theme.value.replace('chain-', '');
                  const palette = CHAIN_PALETTES[paletteKey] || CHAIN_PALETTES['default'];
                  const uniqueChains = [...new Set(window.globalAtoms.map(a => a.chain))].sort();
                  uniqueChains.forEach((ch, idx) => {
                      const c = palette[idx % palette.length];
                      const hex = '#'
                          + Math.round(c.r*255).toString(16).padStart(2,'0')
                          + Math.round(c.g*255).toString(16).padStart(2,'0')
                          + Math.round(c.b*255).toString(16).padStart(2,'0');
                      window.chainColors[ch] = hex;
                  });
                  // Sync picker inputs to new colors
                  const chainSection = document.getElementById('chain-color-section');
                  if (chainSection) {
                      chainSection.querySelectorAll('input[type="color"]').forEach((input, idx) => {
                          const ch = uniqueChains[idx];
                          if (ch) input.value = window.chainColors[ch];
                      });
                  }
                  rebuildColorArray();
              }
              window.clearArt(false);
          };
```
Replace with:
```js
          ui.theme.onchange = () => {
              updateArtPanelBackground();
              window.updateWorkerParams();
              window.clearArt(false);
          };
```

- [ ] **Step 5.2: Remove dead `updateChainColors` function**

Find:
```js
      function updateChainColors() {
          // Legacy: called when chain-* theme selected. Now just triggers rebuildColorArray.
          rebuildColorArray();
      }
```
Delete this function entirely (it is no longer called from anywhere).

- [ ] **Step 5.3: Fix chain-color-section visibility on structure load**

Find the line that sets chain section visibility after load (around line 1704):
```js
                      chainSection.style.display = uniqueChains.length > 0 ? 'block' : 'none';
```
Replace with:
```js
                      const cmodeOnLoad = document.getElementById('art-color-mode')?.value || 'density';
                      chainSection.style.display = (uniqueChains.length > 0 && cmodeOnLoad === 'chain') ? 'block' : 'none';
```

- [ ] **Step 5.4: Verify in browser**

1. Load a structure in Density mode — chain-color-section is hidden. 2. Switch to Chain Colors mode — chain-color-section appears. 3. Switch Chain Palette — per-chain color pickers update to the new palette. 4. Apply a density Style Preset — theme and sliders update, color mode stays on Density. 5. Reload a structure while in Chain Colors mode — chain-color-section remains visible.

- [ ] **Step 5.5: Commit**

```bash
git add index.html
git commit -m "chore: remove chain-* theme handling from theme.onchange, fix chain section visibility on load"
```

---

## Self-Review Against Spec

| Spec Requirement | Covered By |
|---|---|
| Color Mode as master switch | Task 2 (HTML) + Task 3 (JS) |
| Density mode: Theme dropdown density-only, no chain-* | Task 2 |
| Density mode: Style Presets in density panel | Task 2 |
| Chain mode: Chain Palette dropdown | Task 2 (HTML) + Task 3 (onChainPaletteChange) |
| Chain mode: Chain Style (Normal/Neon/Ink/Faded) | Task 2 (HTML) + Task 4 (renderDisplay) |
| Chain mode: Background select (Black/White/Dark) | Task 2 (HTML) + Task 3 (getChainBackground/onBgModeChange) |
| Residue mode: Background select only | Task 2 (HTML) + Task 3 |
| Rendering bug: bg applied to zero-pixels in RGB branch | Task 4 |
| CPU fix: worker early return when paused | Task 1 |
| CPU fix: remove `!isRunning` from shouldSendImage | Task 1 |
