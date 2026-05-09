# ProteinCHAOS — Development Reference (`claude.md`)

> **ProteinCHAOS v1.3** — MD-inspired protein art engine
> Copyright © 2026 Dawid Zyla — GPLv3+
> Single-file application: `index.html` (~175 KB, ~3050 lines)

---

## Table of Contents

1. #1-project-overview
2. #2-architecture
3. #3-physics-simulation-engine-web-worker
4. #4-rendering-pipeline
5. #5-color-system
6. #6-themes
7. #7-style-presets
8. #8-chain-palettes
9. #9-ui-controls--parameters
10. #10-file-format-parsing
11. #11-data-structures
12. #12-topology-builder
13. #13-3d-preview-threejs
14. #14-animation-timeline
15. #15-recording-system
16. #16-export--sharing
17. #17-keyboard-shortcuts
18. #18-touch-input
19. #19-mobile--responsive-layout
20. #20-url-state-encoding
21. #21-external-dependencies
22. #22-key-global-variables
23. #23-development-guidelines
24. #24-known-patterns--gotchas

---

## 1. Project Overview

ProteinCHAOS is a **browser-based, single-file** application that generates artistic visualizations of protein structures inspired by molecular dynamics simulations, long-exposure photography, and generative art. It runs a coarse-grained Langevin dynamics simulation on protein structures and renders the atomic trajectories into a 2D accumulation buffer, producing painterly "chaos art" images.

### Core Concept

- Fetch a protein structure (PDB ID, UniProt/AlphaFold ID, or local file upload)
- Build a coarse-grained model (CA for protein, P for nucleic acids, C1 for glycans, one bead per cofactor)
- Run Langevin dynamics with harmonic bonds, angle restraints, elastic network restraints, and WCA excluded-volume
- Accumulate projected 2D traces into a density buffer
- Tone-map the buffer using ACES filmic mapping with configurable themes
- Display the result with optional bloom, depth-of-field, grain, and depth fog effects

### License

GNU General Public License v3.0 or later.

---

## 2. Architecture

### 2.1 Thread Model

```

Main Thread                          Web Worker

***

Three.js 3D view                    Physics engine
Canvas 2D render         <------>   Force calculation
UI bindings              postMsg    Integration (Langevin)
File parsing                        Accumulation buffer
Tone mapping                        Fade/decay
Bloom/DoF post-process              Noise generation
Export/download                      Spatial hashing (WCA)

```

- **Main thread**: UI, Three.js rendering, 2D canvas tone mapping, post-processing (bloom, DoF, grain), file parsing, export
- **Web Worker**: All physics simulation, force calculation, accumulation buffer updates, fade/decay
- Communication via `postMessage` with transferable ArrayBuffers for zero-copy buffer passing

### 2.2 Message Protocol (Main to Worker)

| Message Type | Payload | Purpose |
|---|---|---|
| `init` | `{atoms, bonds, restraints, angles, colors, width, height}` | Initialize simulation with parsed structure |
| `ensemble` | `{models: [...]}` | Load ghost/ensemble models for multi-model PDB/CIF |
| `colors` | `{data: Float32Array}` | Update per-atom color array (RGB, 0-1) |
| `params` | `{data: {temp, k, speed, zoom, ...}}` | Update simulation/rendering parameters |
| `start` | (none) | Resume simulation |
| `pause` | (none) | Pause simulation |
| `clear` | (none) | Reset accumulation buffer and timers |
| `resize` | `{width, height}` | Resize accumulation buffer |
| `tick` | `{dt}` | Request one simulation frame (dt in ms) |
| `rec_start` | (none) | Enable high-FPS mode for recording |
| `rec_stop` | (none) | Disable high-FPS mode |
| `buf_return` | `{buffer, depthBuffer, gen}` | Return transferred buffers for reuse |

### 2.3 Message Protocol (Worker to Main)

| Message Type | Payload | Purpose |
|---|---|---|
| `state` | `{running: bool}` | Notify simulation running state |
| `update` | `{positions, buffer?, depthBuffer?, time, gen, frames}` | Frame update with optional image buffer |
| `idle` | (none) | Worker has nothing to do |

### 2.4 Buffer Transfer Strategy

- The worker owns the accumulation buffer and copies it to `sentBuffer` before transferring
- Buffers are transferred (zero-copy) via `postMessage(..., [transferables])`
- The main thread returns buffers via `buf_return` after processing
- A `bufGeneration` counter prevents stale buffer usage after resize/mode changes

---

## 3. Physics Simulation Engine (Web Worker)

### 3.1 Constants (`CONF` object)

| Constant | Value | Description |
|---|---|---|
| `FRICTION` | 2.0 | Langevin thermostat friction coefficient |
| `BASE_DT` | 0.0015 | Base integration timestep |
| `BOND_R0` | 1.0 | Default bond rest length (fallback) |
| `ANGLE_BASE_K` | 40.0 | Base angle force constant |
| `WCA_EPSILON` | 1.0 | WCA excluded-volume well depth |
| `PROT_BASE_STIFFNESS` | 32.0 | Protein elastic network base stiffness |
| `PROT_BOND_BASE` | 140.0 | Protein bond force constant base |
| `DNA_BASE_STIFFNESS` | 533.3 | DNA/RNA elastic network base stiffness |
| `DNA_BRACE_BASE` | 46.6 | DNA brace restraint base |
| `DNA_CLAMP_BASE` | 100.0 | DNA clamp restraint base |
| `DNA_SCAFFOLD_BASE` | 60.0 | DNA scaffold restraint base |
| `DNA_ANCHOR_BASE` | 53.3 | DNA anchor restraint base |
| `DNA_PAIR_BASE` | 46.6 | DNA pair restraint base |
| `DNA_TWIST_BASE` | 33.3 | DNA twist restraint base |

### 3.2 Integrator: Langevin Dynamics (BAOAB-like)

The integrator follows a velocity-Verlet / Langevin scheme:

1. **Half-kick**: `v += 0.5 * dt * F * invMass`
2. **Half-drift**: `x += 0.5 * dt * v`
3. **Thermostat (O step)**:
   - `c = exp(-gamma * dt)`
   - `noiseScale = sqrt(T * (1 - c^2) / refMass)`
   - `v = c*v + noiseScale * sqrt(invMass) * flex * gaussian_noise`
   - Includes momentum-biased noise (10-20% momentum correlation based on flexibility)
4. **Half-drift**: `x += 0.5 * dt * v`
5. **Force recalculation**
6. **Half-kick**: `v += 0.5 * dt * F * invMass`
7. **Remove drift** (center-of-mass translation + angular momentum correction)

### 3.3 Force Terms

#### 3.3.1 Harmonic Bonds

```

F = k \* (r - r0)    clamped to +/-1000

```

- Protein bonds: `k = PROT_BOND_BASE * bondScale`
- Nucleic acid bonds: `k = DNA_BASE_STIFFNESS * bondScale`
- `bondScale = max(0.15, sqrt(sliderNorm))` where `sliderNorm = params.k / 20.0`

#### 3.3.2 Angle Restraints (Secondary Structure Preservation)

```

dU/dcos(theta) = kAngle \* (cos(theta) - cos(theta\_0))

```

- `kAngle = ANGLE_BASE_K * bondScale`
- theta_0 is measured from the native structure at parse time
- Applied to consecutive backbone triplets (i, i+1, i+2) within the same chain

#### 3.3.3 Elastic Network Restraints

```

F = k\_local \* (dist - nativeDist)    clamped to +/-500

```

- Protein: `k_local = kProtFlex * stiffness` (stiffness has cubic distance falloff)
- Nucleic acid: `k_local = kClamp * strength`
- `restraintScale = sliderNorm^2`

#### 3.3.4 WCA Excluded Volume (Residue-Specific)

```

F\_WCA = min(24*eps/r^2 \* sigma^2 \* (2*r6i\*r6i - r6i), 500)

```

- sigma_ij = (sigma_i + sigma_j) / 2 (Lorentz combining rule)
- Cutoff: r^2 < sigma^2 * 2^(1/3)
- Per-residue sigma: `sigma = 0.6 + 0.4 * cbrt(mass / 186)`
- Uses flat-array spatial hashing for O(N) neighbor search

#### 3.3.5 Glycan Repulsion

```

F = 80.0 \* (0.8 - r)    for r < 0.8

```

- Applied between glycan atoms and all non-glycan atoms

### 3.4 Spatial Hashing (Flat-Array)

- Cell size: `WCA_CELL_SIZE = maxSigma * 1.3`
- Hash table size: power of 2, minimum `max(16384, 2*nAtoms)`
- Uses FNV-1a-inspired hash: `h = imul(h ^ (coord + 512), 16777619)`
- Open addressing with linear probing
- Rebuilt every 3 ticks (`wcaRebuildTick`)
- Arrays: `hashKeys`, `hashHeads`, `hashNext` (all Int32Array)

### 3.5 Flexibility Model

Per-atom flexibility computed at init:

1. **Contact density**: Count neighbors within radius^2 = 9.0. `baseFlex = max(0.3, 2.0 - nc * 0.17)`
2. **B-factor / pLDDT blending**: PDB B-factors: higher = more flexible. AlphaFold pLDDT: higher = less flexible (auto-detected if all 0-100 and mean > 40). Final: `0.7 * bFlex + 0.3 * baseFlex`
3. **Glycan override**: `flex = max(flex, 1.5)`

### 3.6 Gaussian Noise Cache

- Pre-computed 262144 normal deviates (Box-Muller)
- Circular cursor, regenerated on `init`

### 3.7 Accumulation (Chaos Art Buffer)

Projects 3D to 2D and splatters density:

1. Apply rotation matrix (Euler: pitch/yaw/roll)
2. For each bond/restraint: project, interpolate with random t, apply depth fog, jitter
3. Sub-pixel splatting: 2x2 bilinear (screen <= 2000px) or 3x3 Gaussian (> 2000px)
4. Motion blur: current + midpoint with previous frame (0.5 weight)
5. Ensemble ghosts: 30% alpha
6. Channels: 1 (density) or 3 (chain/residue/SS)

### 3.8 Fade/Decay

- Multiplicative decay in chunks of FADER_BLOCK_SIZE = 524288
- Circular cursor across buffer
- 99.9%+ = no decay (1.0)

---

## 4. Rendering Pipeline

### 4.1 Main Thread Flow

1. Statistical sampling for auto-exposure (p99.5 percentile)
2. displayMax smoothing (fast rise 0.3, slow fall 0.03)
3. Per-pixel: log1p normalize, power curve (sharpness), threshold (gamma), ACES tone map
4. Theme color application or direct RGB
5. Grain overlay (rough/charcoal/graphite only)
6. putImageData
7. Depth of Field (if strength > 0.01)
8. Bloom (3-pass screen blend)
9. Watermarks (if recording)

### 4.2 ACES Tone Mapping

```

acesToneMap(x) = (x\*(2.51*x+0.03)) / (x*(2.43\*x+0.59)+0.14)
ACES\_NORM = 0.8037974683

````

### 4.3 Bloom (3-Pass)

1. 1/4 size, blur width>>6 px, alpha 0.6*bloomVal
2. 1/8 size, blur width>>4 px, alpha 0.3*bloomVal
3. 1/16 size, blur width>>3 px, alpha 0.15*bloomVal
4. Composited via `globalCompositeOperation = 'screen'`

### 4.4 Depth of Field

- Uses latestDepthBuffer (per-pixel z-norm 0-1)
- Focus plane: dof-focus slider. Band width: 0.15
- Max blur: dofStrength * width * 0.02 pixels
- Alpha mask from depth deviation

### 4.5 Grain Overlay

- 256x256 Perlin-like texture (16x16 grid, bicubic)
- Luminance-dependent: `grain = 0.88 + rawG * 0.12 * (1 - lum * 0.75)`

---

## 5. Color System

### 5.1 Color Modes

| Mode | Value | Channels | Description |
|---|---|---|---|
| Density (Theme) | `density` | 1 | Tone-mapped through theme ramps |
| Chain Colors | `chain` | 3 | Per-chain RGB accumulation |
| Residue Properties | `residue` | 3 | By amino acid chemistry |
| Secondary Structure | `ss` | 3 | By helix/sheet/coil |

### 5.2 Residue Property Colors (0-1)

| Property | Residues | RGB |
|---|---|---|
| Hydrophobic | ALA, VAL, ILE, LEU, MET, PHE, TRP, PRO | (0.98, 0.45, 0.09) |
| Polar | SER, THR, CYS, TYR, ASN, GLN | (0.13, 0.83, 0.94) |
| Positive | LYS, ARG | (0.23, 0.51, 0.96) |
| Positive (weak) | HIS | (0.43, 0.55, 0.96) |
| Negative | ASP, GLU | (0.94, 0.27, 0.27) |
| Special | GLY | (0.82, 0.82, 0.82) |
| Unknown | UNK | (0.55, 0.55, 0.55) |
| Nucleic | all | (0.68, 0.54, 0.99) |
| Glycan | all | (0.97, 0.62, 0.07) |
| Cofactor | all | (1.0, 0.42, 0.21) |

### 5.3 Secondary Structure Colors

| Type | Code | RGB |
|---|---|---|
| Helix | H | (0.94, 0.27, 0.51) |
| Sheet | E | (0.24, 0.60, 0.95) |
| Coil | C | (0.75, 0.75, 0.75) |

### 5.4 Chain Styles

| Style | Effect |
|---|---|
| Normal | Direct rendering |
| Neon Glow | 1.8x saturation boost |
| Ink Wash | 25% saturation, inverted on white bg |
| Faded | 60% intensity |

### 5.5 Background Options (Chain/Residue/SS modes)

Black (0,0,0), White (255,255,255), Dark Grey (18,18,18)

---

## 6. Themes

### 6.1 Background Colors

| Theme | BG RGB | Description |
|---|---|---|
| `sunset` | (0,0,0) | Purple-to-gold |
| `fire` | (0,0,0) | Black body radiation |
| `ice` | (0,0,0) | Cyan-blue |
| `ink` | (255,255,255) | Sumi-e ink |
| `rough` | (255,255,255) | Pencil sketch |
| `charcoal` | (30,30,30) | Dark sketch |
| `graphite` | (255,255,255) | Clean pencil |
| `sakura` | (255,255,255) | Cherry blossom |
| `lab` | (255,255,255) | Clinical |
| `editorial` | (255,255,255) | Red/BW |
| `bio` | (5,20,5) | Biohazard green |
| `blueprint` | (10,30,80) | Technical |
| `heme` | (20,5,5) | Deep red |
| `electrostatic` | (10,10,30) | Blue-red field |
| `cyber` | (0,0,0) | Neon |
| `gold` | (20,15,5) | Golden hour |
| `quantum` | (0,0,0) | Purple-teal |
| `viridis` | (68,1,84) | Scientific |
| `magma` | (15,0,30) | Scientific |

White-background themes: ink, graphite, sakura, lab, editorial, rough

### 6.2 Adding a New Theme

1. Add `<option>` to `#art-theme` select
2. Add background to `getThemeBackground()`
3. Add color ramp to `applyTheme()` mapping t (0-1) to RGB (0-255)

---

## 7. Style Presets

| Key | Theme | Zoom | Decay | Gamma | Intensity | Jitter |
|---|---|---|---|---|---|---|
| `editorial-clean` | ink | 45 | 98.5 | 0.7 | 45 | 0.2 |
| `editorial-dark` | viridis | 40 | 99.0 | 0.6 | 50 | 0.3 |
| `blueprint` | blueprint | 50 | 97.0 | 0.5 | 60 | 0.1 |
| `lab` | lab | 45 | 98.0 | 0.6 | 50 | 0.2 |
| `chaos-fire` | magma | 30 | 96.0 | 0.4 | 80 | 1.5 |
| `chaos-gold` | gold | 35 | 97.5 | 0.5 | 60 | 1.0 |
| `sketch` | rough | 40 | 92.0 | 0.6 | 70 | 0.8 |
| `cyber` | cyber | 35 | 98.5 | 0.7 | 55 | 0.5 |

---

## 8. Chain Palettes

| Palette | Colors (hex) | Count |
|---|---|---|
| `default` | #93c5fd, #6ee7b7, #fbcfe8, #fde68a, #c084fc, #fca5a5, #67e8f9, #e5e7eb | 8 |
| `spectral` | #9e0142 through #5e4fa2 | 10 |
| `neon` | #ff00ff, #00ffff, #ffff00, #ff0000, #00ff00, #0000ff | 6 |
| `retro` | #264653, #2a9d8f, #e9c46a, #f4a261, #e76f51 | 5 |
| `earth` | #582f0e through #656d4a | 8 |
| `vivid` | #ff3b5c, #ff9f0a, #30d158, #64d2ff, #bf5af2, #ff6961, #ffd60a, #0a84ff | 8 |
| `publication` | #222222, #555555, #888888, #bbbbbb, #1a6b8a, #8a3324, #4a7c59, #7a4f2e | 8 |
| `wong` | #e69f00, #56b4e9, #009e73, #f0e442, #0072b2, #d55e00, #cc79a7, #000000 | 8 |
| `pastel` | #ffb3ba, #ffdfba, #ffffba, #baffc9, #bae1ff, #d4baff, #ffbaee, #c9f5f5 | 8 |

---

## 9. UI Controls & Parameters

### 9.1 Source Data

| Control | ID | Type | Default |
|---|---|---|---|
| Model Glycans | `model-glycans` | checkbox | checked |
| Model Cofactors | `model-cofactors` | checkbox | checked |
| Bio Assembly | `model-assembly` | checkbox | unchecked (hidden if no BIOMT) |
| Show Ensemble | `model-ensemble` | checkbox | unchecked (hidden if single model) |
| PDB ID | `pdb-id` | text | "2HHB" |
| File Upload | `file-upload` | file | .pdb, .ent, .cif, .mmcif |

### 9.2 Physics

| Control | ID | Min | Max | Step | Default |
|---|---|---|---|---|---|
| Temperature | `temp-slider` | 0 | 2.5 | 0.01 | 0.40 |
| Stiffness | `k-restraint` | 1.0 | 100.0 | 0.1 | 20.0 |
| Speed | `sim-speed` | 0.1 | 5.0 | 0.1 | 1.00 |

### 9.3 Camera

| Control | ID | Min | Max | Step | Default |
|---|---|---|---|---|---|
| Pitch | `cam-rot-x` | 0 | 360 | 1 | 0 |
| Yaw | `cam-rot-y` | 0 | 360 | 1 | 0 |
| Roll | `cam-rot-z` | 0 | 360 | 1 | 0 |
| Pan X | `cam-pan-x` | -400 | 400 | 1 | 0 |
| Pan Y | `cam-pan-y` | -400 | 400 | 1 | 0 |
| Zoom | `art-zoom-control` | 5 | 200 | 1 | 35 (auto-fit) |
| Auto-Spin | `spin-toggle` | checkbox | | | off |
| Spin Speed | `spin-speed` | -10 | 10 | 0.1 | 2 |

Presets: Front (0,0,0), Top (90,0,0), Side (0,90,0)

### 9.4 Rendering Sliders

| Control | ID | Min | Max | Step | Default | Notes |
|---|---|---|---|---|---|---|
| Trace Persistence | `art-decay` | 80 | 100 | 0.1 | 98 | 99.9+ = INF |
| Density Threshold | `art-gamma` | 0.0 | 0.9 | 0.01 | 0.00 | |
| Trail Sharpness | `art-softness` | 0.3 | 5.0 | 0.1 | 2.0 | Power exponent |
| Texture/Noise | `art-intensity` | 1 | 200 | 1 | 40 | |
| Jitter / Chaos | `art-jitter` | 0 | 5.0 | 0.1 | 0.8 | |
| Bloom / Glow | `art-bloom` | 0 | 0.5 | 0.01 | 0.15 | |
| Depth Fog | `art-depthfog` | 0 | 1.0 | 0.01 | 0.30 | Clears art |
| DoF Focus | `art-dof-focus` | 0 | 1 | 0.01 | 0.50 | |
| DoF Strength | `art-dof-strength` | 0 | 1 | 0.01 | 0.00 | 0 = off |

### 9.5 Other Controls

- **Render Resolution**: Toggle between Fast (screen) and High Res (4K target at 2160px height, max 7680x4320)
- **Draw Source**: `trace` (backbone bonds) or `restraints` (elastic web)
- **Auto snapshots**: toggle + interval in frames
- **History**: max 40 snapshots, slider to restore
- **Timeline**: Duration 1-120s, scrubber 0-100%

### 9.6 Animatable Parameters (16)

`temp`, `k`, `speed`, `zoom`, `decay`, `gamma`, `softness`, `intensity`, `jitter`, `bloom`, `depthfog`, `rotX`, `rotY`, `rotZ`, `panX`, `panY`

---

## 10. File Format Parsing

### 10.1 PDB Format

- ATOM/HETATM: fixed-width columns for name, residue, chain, resNum, xyz, B-factor
- HELIX/SHEET: SS assignments
- LINK/SSBOND: covalent/disulfide bonds
- MODEL/ENDMDL: multi-model ensembles
- REMARK 350 BIOMT: biological assembly matrices

### 10.2 mmCIF Format

Custom `parseCIFLoops()` parser:
- Categories: `_atom_site`, `_pdbx_struct_oper_list`, `_struct_conn`
- Handles quoted strings, semicolon multi-line, alternate conformations (keeps 'A' only)
- Prefers auth_* over label_* columns

### 10.3 Atom Extraction

| Type | Atom Name | Condition |
|---|---|---|
| Protein | CA | Not glycan/cofactor |
| Nucleic acid | P | DNA/RNA residues |
| Glycan | C1 | NAG, MAN, BMA, etc. |
| Cofactor | First per unique key | HEM, ATP, ions, etc. |

### 10.4 Coordinates

All divided by 4 at parse. Centered. Auto-zoom fitted.

### 10.5 Fetch Strategy

- 4-char ID: RCSB `.pdb` then `.cif` (3 URLs)
- Other: AlphaFold v6, v4, v3 (3 URLs)
- 5-second timeout per attempt

### 10.6 Recognized Names

**Glycans**: NAG, MAN, BMA, FUC, GAL, GLC, SIA, NDG, FUL

**Cofactors**: HEM, HEC, ATP, ADP, AMP, GTP, GDP, GMP, NAD, NAP, NDP, FAD, FMN, COA, ACO, PLP, SF4, FES, ZN, MG, CA, FE, MN, CU, CO, NI, CLR, PLM, OLA, RET

---

## 11. Data Structures

### 11.1 Atom

```javascript
{ x, y, z, isNucleic, isGlycan, isCofactor, isProtein,
  resName, simMass, visualWeight, chain, resNum, bfactor,
  sigma, ss, isAlphaFold }
````

### 11.2 Bond

```javascript
{ i, j, visualWeight, d? }
```

### 11.3 Restraint

```javascript
{ i, j, nativeDist, stiffness?, strength?, visualWeight }
```

### 11.4 Angle

```javascript
{ i, j (central), k, theta0 }
```

### 11.5 Masses (Da)

Amino acids: ALA=71, ARG=156, ASN=114, ASP=115, CYS=103, GLN=128, GLU=129, GLY=57, HIS=137, ILE=113, LEU=113, LYS=128, MET=131, PHE=147, PRO=97, SER=87, THR=101, TRP=186, TYR=163, VAL=99

Nucleotides: DA=313, DT=304, DG=329, DC=289, A=313, T=304, G=329, C=289, U=306

Cofactors: HEM=616, ATP=507, ADP=427, GTP=523, NAD=663, FAD=785, FMN=456, COA=767, PLP=247, SF4=352, ZN=65, MG=24, CA=40, FE=56, default=200

### 11.6 Visual Weight

*   Protein/nucleic: `min(1.0, 0.2 + (mass / 330) * 0.8)`
*   Glycans: 0.6
*   Cofactors: `min(1.0, 0.4 + (mass / 800) * 0.6)`

***

## 12. Topology Builder

### 12.1 Bond Generation Order

1.  **Backbone**: sequential atoms, same chain, distance < 3.0
2.  **LINK/SSBOND**: from PDB/CIF records
3.  **Glycan anchoring**: per disconnected cluster, closest protein atom (scoring: root sugar + target residue preference)
4.  **Cofactor anchoring**: nearest non-cofactor within 5.0
5.  Uses **Union-Find** for connectivity tracking

### 12.2 Angles

Consecutive backbone triplets (i, i+1, i+2), same chain, non-glycan/cofactor. Native angle theta\_0 measured at parse.

### 12.3 Elastic Network

*   Cell-based spatial search (cell size 5.0)
*   Protein: r < 8.0, weight = (1 - r/8)^3
*   Nucleic: r < 12.0, strength = 1.0
*   Glycans excluded

### 12.4 SS Detection Fallback

If no HELIX/SHEET records: angle < 105 = H, 105-140 = E, else C

### 12.5 Biological Assembly

*   BIOMT matrices (3x4, Float64Array)
*   Duplicates atoms with transformed coords
*   Chain suffix `_N`
*   Warns if > 25000 residues

### 12.6 Ensemble Support

*   Multiple models from NMR/ensemble files
*   Model 1 = dynamic, others = static ghosts at 30% alpha
*   Sent via `ensemble` message

***

## 13. 3D Preview (Three.js)

### 13.1 Setup

*   Three.js 0.160.0 (ES module CDN)
*   PerspectiveCamera: FOV=45, near=0.1, far=1000
*   ArcballControls (gizmos hidden, damping)
*   AmbientLight: intensity=1.4
*   Scene bg: #12131f, fog: #12131f (40-220)

### 13.2 Post-Processing

1.  RenderPass
2.  SSAOPass: kernelRadius=1.5, minDist=0.005, maxDist=0.15
3.  UnrealBloomPass: threshold=0.6, strength=0.15, radius=0.3
4.  OutputPass

### 13.3 Instanced Meshes

*   Atoms: SphereGeometry(0.4, 24, 24), MeshBasicMaterial
*   Bonds: CylinderGeometry(0.1, 0.1, 1, 8), rotated PI/2 on X
*   Per-instance colors via setColorAt()
*   Updated each frame from worker positions

### 13.4 3D Color Modes

*   Chain: `window.chainColors[atom.chain]`
*   Atom: type-based palettes (VIEW\_PROTEIN\_COLORS: 5 blues/greens/pinks; VIEW\_DNA\_COLORS: 3 purples/cyans; VIEW\_GLYCAN\_COLORS: 3 ambers; VIEW\_COFACTOR\_COLORS: 3 oranges/reds)

***

## 14. Animation Timeline

*   Keyframes store all animatable params at time (seconds)
*   First KF at t=0, subsequent user-prompted
*   Interpolation: Hermite smoothstep `t^2 * (3 - 2t)`
*   Duration: 1-120s (default 10)
*   Playback auto-starts simulation
*   Manual scrubber for preview

***

## 15. Recording System

### Free Recording

*   5 seconds, canvas.captureStream(30) + MediaRecorder
*   Codec: VP9 > WebM > MP4, 8 Mbps
*   Watermarks included

### Boomerang

*   Captures 75 frames forward as ImageData
*   Replays in reverse at \~30fps
*   Auto-switches to Fast res if width > 2500

***

## 16. Export & Sharing

| Action      | Output        | Details                                                               |
| ----------- | ------------- | --------------------------------------------------------------------- |
| Download HD | PNG           | Full res + watermarks (PDB ID top-right, "ProteinCHAOS" bottom-right) |
| Export SVG  | SVG           | Vector lines from projected bonds, theme/chain colors                 |
| Copy Img    | Clipboard PNG | Canvas to clipboard API                                               |
| Share       | URL           | All settings encoded as query params                                  |

Filename pattern: `chaos_{pdbId}_{theme}.{ext}`

***

## 17. Keyboard Shortcuts

| Key                  | Action            |
| -------------------- | ----------------- |
| Space                | Toggle play/pause |
| S                    | Save snapshot     |
| C                    | Clear canvas      |
| D                    | Download HD       |
| Enter (in PDB input) | Fetch structure   |

All except Enter require focus NOT on input/select/textarea, no Ctrl/Meta.

***

## 18. Touch Input

Two-finger on art panel:

*   **Pinch**: Zoom (delta \* 0.3)
*   **Drag**: Pan (midpoint delta)
*   `passive: false`, `touch-action: none`

***

## 19. Mobile Layout

Breakpoint: 768px

| Property      | Desktop    | Mobile              |
| ------------- | ---------- | ------------------- |
| Direction     | Row        | Column              |
| Main view     | Flex width | 100%, 55dvh, top    |
| Sidebar       | 360px      | 100%, 45dvh, bottom |
| 3D height     | 40%        | 30%                 |
| Touch targets | Default    | Min 44px            |
| Slider thumbs | 14px       | 24px                |

***

## 20. URL State Encoding

Params: `pdb`, `theme`, `temp`, `k`, `speed`, `zoom`, `decay`, `gamma`, `softness`, `intensity`, `jitter`, `bloom`, `depthfog`, `source`, `cmode`, `rx`, `ry`, `rz`, `px`, `py`

Auto-fetches on load if `pdb` or `af` present.

***

## 21. External Dependencies

| Library              | Version | Purpose                                                                                                    |
| -------------------- | ------- | ---------------------------------------------------------------------------------------------------------- |
| Three.js             | 0.160.0 | 3D rendering + addons (ArcballControls, EffectComposer, SSAOPass, UnrealBloomPass, OutputPass, RenderPass) |
| Tailwind CSS Browser | 4.x     | Runtime utility CSS                                                                                        |

No build tools required. Single HTML file.

***

## 22. Key Global Variables

| Variable             | Type            | Description                                                                    |
| -------------------- | --------------- | ------------------------------------------------------------------------------ |
| `window.globalAtoms` | Array           | Parsed atoms                                                                   |
| `window.globalBonds` | Array           | Bond list                                                                      |
| `window.chainColors` | Object          | chain -> hex color                                                             |
| `worker`             | Worker          | Physics worker                                                                 |
| `state`              | Object          | `{isRunning, hasStructure, hasStarted, isLoading, workerRunning, isRecording}` |
| `latestBuffer`       | Float32Array    | Latest accumulation buffer                                                     |
| `latestDepthBuffer`  | Float32Array    | Latest depth buffer                                                            |
| `historyStack`       | Float32Array\[] | Max 40 snapshots                                                               |
| `displayMax`         | number          | Auto-exposure max (starts 0.5)                                                 |
| `renderScale`        | number          | Resolution multiplier                                                          |
| `artWidth/artHeight` | number          | Canvas size (max 7680x4320)                                                    |
| `currentPdbId`       | string          | Current ID (default "1CDW")                                                    |
| `keyframes`          | Array           | Timeline keyframes                                                             |
| `frameCounter`       | number          | Total frames                                                                   |

***

## 23. Development Guidelines

### Adding a Theme

1.  Add `<option>` to `#art-theme`
2.  Add bg to `getThemeBackground()`
3.  Add ramp to `applyTheme()`

### Adding a Color Mode

1.  Add `<option>` to `#art-color-mode`
2.  Update `onColorModeChange()` panel visibility
3.  Update `rebuildColorArray()` for per-atom RGB
4.  Add sub-panel if needed

### Adding Physics Forces

1.  Add to `calcForces()` in worker string
2.  Add per-atom arrays in `init` handler
3.  Apply force clamping

### Adding UI Controls

1.  Add HTML element
2.  Add to `ui` object
3.  Wire oninput to `updateWorkerParams()`
4.  Add to worker params
5.  Add to URL encoding + ANIMATABLE\_PARAMS if needed

### Performance Notes

*   Variable substeps: max(1, min(40, round(dt\*speed/1.5)))
*   Frame skip: 1-3 based on dt
*   Spatial hash rebuild every 3 ticks
*   Zero-copy buffer transfer
*   Noise cache: 262144 pre-computed
*   Force clamp: bonds +/-1000, restraints/WCA +/-500

***

## 24. Known Patterns & Gotchas

1.  **bufGeneration**: Increments on resize/mode change. Stale buffers discarded.
2.  **Channel switching**: density (1ch) vs chain/residue/SS (3ch) reallocates worker buffer.
3.  **Coordinate scale**: All coords /4 at parse. Zoom compensates.
4.  **Pan scaling**: UI values \* renderScale before sending to worker.
5.  **Decay >= 99.9**: Factor set to 1.0, displayed as "INF".
6.  **Tick flow control**: One tick in-flight at a time (workerTickInFlight flag).
7.  **History viewing**: Pauses live buffer updates when scrubbed backward.
8.  **Auto-spin**: Modifies cam-rot-y directly each frame.
9.  **WebGL recovery**: Three.js handles context loss/restore. 2D canvas unaffected.
10. **Window functions**: Many exposed on window for HTML onclick handlers (fetchPDB, toggleSim, clearArt, saveSnapshot, downloadHighRes, exportSVG, etc.)
11. **`||` pattern**: Used throughout as null coalescing and zero-guard.
12. **Watermarks**: Only during recording and HD export, not live preview.
13. **3D view colors**: Independent selector from art canvas color mode.

