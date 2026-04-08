# Theme & Color Mode Redesign

**Date:** 2026-04-07  
**Status:** Approved  

## Problem

The current UI has two overlapping controls that create broken and confusing combinations:

1. **Theme dropdown** (`#art-theme`) — contains both density themes (fire, ink, blueprint…) AND chain palette names (Chain: Default, Chain: Spectral…)
2. **Color Mode dropdown** (`#art-color-mode`) — separately switches density / chain / residue rendering

This allows nonsensical combinations (e.g. blueprint theme + chain color mode) and contains a rendering bug: in chain/residue mode, the theme's background color is computed but never applied to zero-value pixels — so the background is always black regardless of theme.

## Design

### Color Mode as Master Switch

`#art-color-mode` stays at the top of the Style section with three options:
- **Density** — density/heatmap rendering using a theme
- **Chain Colors** — per-chain RGB accumulation
- **Residue Properties** — per-residue fixed colors

Switching mode shows/hides sub-panels relevant to that mode. Impossible combinations are prevented by design.

---

### Mode: Density

**Controls shown:** Theme dropdown + Style Presets  
**Controls hidden:** Chain palette, chain style, per-chain pickers, background toggle

The Theme dropdown contains **only** density themes (19 total):
sunset, fire, ice, ink, rough, charcoal, graphite, sakura, lab, editorial, heme, electrostatic, blueprint, bio, cyber, gold, quantum, viridis, magma

Background is baked into each theme (white for ink/graphite/sakura etc., dark for fire/viridis etc.) — no separate background control needed.

The `chain-*` options are removed from the Theme dropdown entirely.

Style Presets work unchanged — all 8 existing presets are density-mode presets.

---

### Mode: Chain Colors

**Controls shown:** Chain Palette dropdown + Chain Style dropdown + per-chain color pickers + Background toggle  
**Controls hidden:** Theme dropdown, Style Presets

**Chain Palette dropdown** (replaces chain-* theme options):
- Default, Spectral, Neon, Retro, Earth, Vivid, Publication, Wong (colorblind-safe), Pastel

**Chain Style dropdown** (new — applied in the RGB render branch of `renderDisplay()`):
- **Normal** — raw chain colors, accumulation tone-mapped as-is
- **Neon Glow** — saturation boosted; colors bleed into dark background
- **Ink Wash** — chain colors desaturated toward greyscale (best on white background)
- **Faded** — lower intensity, soft/translucent look

**Background toggle**: Black / White / Dark Grey  
Maps to `{r,g,b}`: `{0,0,0}` / `{255,255,255}` / `{18,18,18}`

---

### Mode: Residue Properties

**Controls shown:** Background toggle only  
**Controls hidden:** Theme dropdown, Chain Palette, Chain Style, per-chain pickers, Style Presets

Residue colors (hydrophobic=orange, polar=cyan, positive=blue, negative=red, glycine=grey) are fixed — no user control needed.

**Background toggle**: Black / White / Dark Grey (same as chain mode)

---

### Rendering Bug Fix

In `renderDisplay()`, the RGB branch (chain/residue modes) currently ignores the `bg` variable. Fix: when pixel accumulation is zero, fill with `bg.r, bg.g, bg.b` instead of always `0,0,0`.

```
// Before (broken):
data[idx] = r * 255; data[idx+1] = g * 255; data[idx+2] = b * 255; data[idx+3] = 255;

// After:
if (r === 0 && g === 0 && b === 0) {
    data[idx] = bg.r; data[idx+1] = bg.g; data[idx+2] = bg.b;
} else {
    data[idx] = r * 255; data[idx+1] = g * 255; data[idx+2] = b * 255;
}
data[idx+3] = 255;
```

A new helper `getChainBackground(bgMode)` maps the toggle value to `{r,g,b}` and is called instead of `getThemeBackground` when in chain/residue mode.

---

### Chain Style Implementation

Chain styles are applied inside the RGB render branch of `renderDisplay()`, after tone-mapping each channel. No worker changes needed.

**Normal:** `r * 255, g * 255, b * 255` (current behavior)

**Neon Glow:** Boost saturation by increasing distance from grey, then clamp:
```js
const avg = (r + g + b) / 3;
r = Math.min(1, avg + (r - avg) * 1.8);
// same for g, b
```
Best on Black background.

**Ink Wash:** Lerp each channel toward luminance (desaturate), then invert for white bg:
```js
const lum = 0.299*r + 0.587*g + 0.114*b;
r = lum + (r - lum) * 0.25; // 75% desaturated
// Optionally invert: r = 1 - r  (for white bg)
```

**Faded:** Scale tone-mapped values down to ~60% intensity:
```js
r = r * 0.6; g = g * 0.6; b = b * 0.6;
```

---

## UI Structure (after change)

```
[ Color Mode ▼ ]  Density | Chain Colors | Residue Properties

--- Density mode ---
[ Theme ▼ ]       sunset / fire / ice / ink / ... (19 options)
[ Style Preset ▼ ] Custom / Journal Figure / ...

--- Chain Colors mode ---
[ Chain Palette ▼ ] Default / Spectral / Neon / ...
[ Chain Style ▼ ]   Normal / Neon Glow / Ink Wash / Faded
[ per-chain color pickers ]
[ Background ]  ● Black  ○ White  ○ Dark Grey

--- Residue Properties mode ---
[ Background ]  ● Black  ○ White  ○ Dark Grey
```

---

## CPU Fix: Idle Worker Drain

**Problem:** When paused, the main thread still posts `tick` messages to the worker at 60 fps via `requestAnimationFrame`. In the worker, `shouldSendImage` includes `|| !isRunning`, so a full buffer transfer happens every frame even when nothing has changed — keeping the CPU pinned.

**Fix — worker side (inside `workerCode` string):**

1. Add early-return when paused and no frame needed:
```js
} else if (msg.type === 'tick') {
    if (nAtoms === 0) { postMessage({ type: 'idle' }); return; }
    if (!isRunning && !needsFrame) { postMessage({ type: 'idle' }); return; } // NEW
```

2. Remove `!isRunning` from `shouldSendImage`:
```js
// Before:
const shouldSendImage = forceHighFPS || (frameCounter % displayFrameSkip === 0) || !isRunning || needsFrame;
// After:
const shouldSendImage = forceHighFPS || (frameCounter % displayFrameSkip === 0) || needsFrame;
```

The pause handler already sets `needsFrame = true`, so one final frame is flushed after pause. After that, all subsequent ticks are short-circuited.

**Fix — main thread:** Stop posting tick messages when paused and no structure interaction is needed:
```js
// Before:
if (worker && state.hasStructure && !workerTickInFlight) {
// After:
if (worker && state.hasStructure && !workerTickInFlight && (state.workerRunning || state.needsWorkerDrain)) {
```

Track `state.needsWorkerDrain = true` when pause is requested; clear it when worker returns the first idle after pause.

---

## Files Changed

- `index.html` only (single-file project, no build system)

## Out of Scope

- No changes to the 3D preview color mode (`#view-color-mode`)
- No changes to Style Presets for chain/residue (presets remain density-only)
- No new worker message types (uses existing `idle` return)
