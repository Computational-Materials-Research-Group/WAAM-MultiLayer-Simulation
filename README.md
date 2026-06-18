# WAAM-MultiLayer-Simulation

<p align="center">
  <img src="https://img.shields.io/badge/FreeFEM++-Simulation-blue?style=for-the-badge&logo=gnu&logoColor=white"/>
  <img src="https://img.shields.io/badge/Mild%20Steel-S235%2FA36-darkgreen?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/3D-Transient%20Heat%20Transfer-orange?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/26%20Layers-Element%20Activation-red?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Goldak-Double%20Ellipsoid-purple?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/ParaView-VTK%20Export-green?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/License-CC%20BY--NC%204.0-lightgrey?style=for-the-badge"/>
</p>

<p align="center">
  A fully 3D transient heat transfer simulation of <b>Wire Arc Additive Manufacturing (WAAM)</b>
  implemented in FreeFEM++. A vertical steel wall is built layer by layer
  on a flat substrate using a <b>moving Goldak double-ellipsoid heat source</b> and
  <b>quiet element activation</b>, producing layer-by-layer temperature history,
  cumulative thermal strain, and residual stress fields.
</p>

<img width="1008" height="772" alt="waam" src="https://github.com/user-attachments/assets/be9a0ce6-ec00-4f16-af29-be3b1b6bac4e" />

---

## Concept

WAAM deposits metal wire layer by layer using an electric arc as the heat source.
This simulation models a 26-layer steel wall built on a flat substrate:

| Aspect | Description |
|---|---|
| Heat source | Goldak double-ellipsoid moving at vscan = 10 mm/s |
| Deposition | 26 layers x 3mm on 200x100x10mm base plate |
| Activation | Quiet element method — layers activate as torch arrives |
| Output | Temperature history, thermal strain, residual stress |

The deposit is a 200x8x78mm steel wall, built in 26 passes of 3mm each
on a 200x100x10mm steel base plate.

---

## Physics

The simulation solves the **3D transient heat equation** with element activation:

- **Conduction-dominated heat transfer** — `rho*Cp(T)*dT/dt = div(k(T)*grad T) + Q_Goldak(x,t)`
- **Temperature-dependent properties** — `k(T)` and `Cp(T)` for mild steel S235 including Curie peak at 720C
- **Moving Goldak source** — double-ellipsoid advances at `vscan = 10 mm/s`, resets each layer
- **Quiet element activation** — inactive layers have `k_eff = 1e-4 * k_real`, `Cp_eff = 1e-4 * Cp_real`
- **Progressive activation** — nodes activate as torch arrives at their x-position
- **Source masking** — `Q_Goldak * psiActive` ensures arc only heats existing material
- **Cumulative thermal strain** — `epsTh += alpha * dT` each step for mechanical post-processing

---

## Geometry

```
        z
        |  Layer 26  <- last deposited  (z = 85-88 mm)
        |  Layer 25
        |  ...
        |  Layer  2  (z = 13-16 mm)
 HzBase |  Layer  1  <- first deposited (z = 10-13 mm)
        |
      0 +----------------------------------------------- x
        |           BASE PLATE (substrate)
        |   200 mm long x 100 mm wide x 10 mm thick

  Deposit: 200 mm long x 8 mm wide x 78 mm tall (26 x 3 mm)
  Centred at y = 50 mm (mid-width of base plate)
  Torch traverses x = 0 to 200 mm for each layer
```

---

## Material Properties — Mild Steel S235 / A36

| Property | Symbol | Value | Unit |
|---|---|---|---|
| Density | rho | 7850 | kg/m3 |
| Conductivity (20C) | k | 60 | W/mK |
| Conductivity (800C) | k | 38 | W/mK |
| Conductivity (liquid) | k_l | 27 | W/mK |
| Specific heat (20C) | Cp | 440 | J/kgK |
| Specific heat (Curie peak ~720C) | Cp | 5000 | J/kgK |
| Specific heat (liquid) | Cp_l | 800 | J/kgK |
| Solidus | T_sol | 1450 | C |
| Liquidus | T_liq | 1520 | C |
| Youngs modulus (20C) | E | 210 | GPa |
| Poisson ratio | nu | 0.30 | -- |

---

## Process Parameters (GMAW/MIG)

| Parameter | Symbol | Value | Unit |
|---|---|---|---|
| Welding current | I | 200 | A |
| Voltage | U | 24 | V |
| Efficiency | eta | 0.8 | -- |
| Power | P | 3840 | W |
| Traverse speed | v_scan | 10 | mm/s |
| Layer height | h_bead | 3 | mm |
| Bead width | w_bead | 8 | mm |
| Number of layers | N | 26 | -- |
| Time per pass | t_pass | 20 | s |
| Inter-layer wait | t_wait | 60 | s |
| Total simulation time | t_total | 2080 | s |
| Ambient temperature | T_amb | 25 | C |

---

## Goldak Double-Ellipsoid Parameters

```
  Q(x,y,z,t) = [Q0f * exp(-3*(x-xT)^2/af^2) * frontFrac
              +  Q0r * exp(-3*(x-xT)^2/ar^2) * rearFrac]
              * exp(-3*(y-yT)^2/bG^2)
              * exp(-3*(z-zT)^2/cG^2)
              * psiActive

  frontFrac = 0.5*(1 + tanh((x-xT)/eps))    smooth front/rear split
  rearFrac  = 1 - frontFrac
```

| Parameter | Symbol | Value | Notes |
|---|---|---|---|
| Front semi-axis | af | 8 mm | >= hx element size (6.7mm) |
| Rear semi-axis | ar | 20 mm | Elongated wake |
| Half-width Y | bG | 7 mm | >= hy element size (5mm) |
| Depth Z | cG | 4 mm | Weld penetration depth |
| Front fraction | ff | 0.6 | ff + fr = 2 |
| Rear fraction | fr | 1.4 | |

---

## Element Activation (Quiet Element Method)

```
  t < t_activate(iL):   k_eff = 1e-4 * k_real,  Cp_eff = 1e-4 * Cp_real
  t >= t_activate(iL):  k_eff = k_real,          Cp_eff = Cp_real

  Progressive activation: node at x_node activates when xTorch >= x_node - af
  Source masking:         Q_Goldak * psiActive  (no heat to non-existent material)

  Layer iL activation time: t_act = iL * (t_pass + t_wait)
  Source z-centre rises:    z_mid = HzBase + (iL + 0.5) * hBead
```

---

## Mesh

| Component | Nx | Ny | Nz | Element size |
|---|---|---|---|---|
| Base plate | 30 | 20 | 8 | 6.7 x 5.0 x 1.25 mm |
| Deposit zone | 30 | 6 | 26 | 6.7 x 1.3 x 3.0 mm |
| Combined | -- | -- | -- | ~11 687 nodes, ~56 880 tets |

Linear `movemesh3` scaling only — no trigonometric functions in `transfo` (required for FreeFEM++ v4.15 Windows compatibility).

---

## Adaptive Timestepping

| Phase | dt | Reason |
|---|---|---|
| Welding pass | 0.333 s | vscan * dt = 3.3mm ~ 0.5 * hx (stability) |
| Inter-layer wait | 10 s | 6x coarser — no source active |
| Final cooling | 120 s | Field near-uniform, coarse dt sufficient |

---

## Output Files — `D:\freefem++\WAAM_MultiLayer_Simulation\`

| File | Description |
|---|---|
| `waam.pvd` | Master animation — open this in ParaView |
| `T_0000.vtu` | Frame 0 (initial state, substrate only) |
| `T_NNNN.vtu` | Temperature + Activation fields per frame |
| `waam_mech.vtu` | Final displacement and von Mises stress |
| `waam_data.csv` | Layer-by-layer: time, Tmax, inter-pass T, epsTh |

### CSV Columns

| Column | Description | Unit |
|---|---|---|
| time_s | Physical simulation time | s |
| layer | Current layer number (1-26) | -- |
| phase | WELDING or INTERPASS | -- |
| xTorch_mm | Torch x-position | mm |
| zLayer_mm | Z-centre of current layer | mm |
| Tmax_C | Maximum temperature in domain | C |
| TinterPass_C | Peak temperature at inter-pass snapshot | C |
| epsTh_max | Maximum cumulative thermal strain | -- |

---

## Repository Structure

```
waam_multilayer.edp                      # Main FreeFEM++ simulation script
README.md                            # This file

D:\freefem++\WAAM_MultiLayer_Simulation\
├── waam.pvd                         # Master animation (open first)
├── waam_data.csv                    # Layer-by-layer diagnostics
├── T_0000.vtu                       # Initial state
├── T_0001.vtu                       # After first weld step
├── ...
├── T_NNNN.vtu                       # Final cooling frame
└── waam_mech.vtu                    # Residual stress and distortion
```

---

## How to Run

### Requirements
- FreeFEM++ v4.10 or later: https://freefem.org
- ParaView v5.x or later: https://www.paraview.org

### Step 1 — Run the simulation

```bash
FreeFem++ waam_multilayer.edp
```

The script will:
1. Build two cube meshes (base plate + deposit zone) and concatenate them
2. Initialise substrate as active, all deposit layers as quiet
3. For each of 26 layers:
   - Progressively activate nodes as torch arrives
   - Run welding pass (torch x = 0 to 200mm, dt = 0.333s)
   - Run inter-layer cooling (60s, dt = 10s)
   - Save VTU frames with `TempC` and `Activation` fields
4. Run final 40-minute cooling
5. Solve elastic mechanical problem using cumulative thermal strain
6. Save `waam_mech.vtu` with distortion and von Mises stress

Console output example:
```
==========================================
 T-JOINT AS WAAM PROCESS
 Substrate: 200x100x10mm
 Deposit: 26 layers x 3mm = 78mm total
 P=3840W  v=10mm/s  tPass=20s  tWait=60s
==========================================
 Combined: 11687 nodes  56880 tets
 === WAAM DEPOSITION ===
--- Layer 1/26  z=[10,13]mm  t=0s ---
  L1 ws=0   x=0mm    Tmax=1847C
  L1 ws=3   x=9.99mm  Tmax=1923C
  L1 ws=60  x=199mm   Tmax=1812C
  -> Inter-pass cool: Tmax=412C  t=80s
--- Layer 2/26  z=[13,16]mm  t=80s ---
...
```

### Step 2 — Open in ParaView

1. `File > Open` → navigate to `D:\freefem++\WAAM_MultiLayer_Simulation\`
2. Select `waam.pvd` → OK → Apply

### Step 3 — Hide inactive layers

Apply Threshold filter to show only deposited material:

```
Filters > Search > Threshold
  Scalars:  Activation
  Minimum:  0.5
  Maximum:  1.0
Click Apply
```

Then color by `TempC` and press **Play**.

### Step 4 — Visualise

**Option A — Layer-by-layer buildup**
```
Color by:     Activation
Colormap:     Cool to Warm
Range:        0 to 1
Press Play -> watch each layer activate as torch traverses
```

**Option B — Temperature field (like Abaqus NT11)**
```
Color by:     TempC
Colormap:     Rainbow or Inferno
Range:        25 to 2000
Press Play -> Goldak hot spot moves right across each layer,
             then cools before next layer begins
```

**Option C — Cross-section through deposit**
```
Filters > Clip
  Normal:  Y axis
  Origin:  y = 50mm (centre of deposit)
-> See temperature gradient through wall thickness
-> Hot at top (current layer), cool below (previous layers)
```

**Option D — Residual stress and distortion**
```
Open waam_mech.vtu (separate file)
Color by SigMises -> residual stress in MPa
Filters > Warp By Vector (Ux,Uy,Uz) scale = 50x -> distortion shape
-> Base plate bows away from deposit due to thermal contraction
```

**Option E — Post-process CSV**
```python
import pandas as pd
import matplotlib.pyplot as plt

df = pd.read_csv('waam_data.csv')
weld = df[df['phase']=='WELDING']
inter = df[df['phase']=='INTERPASS']

plt.figure(figsize=(12,4))
plt.plot(weld['time_s'], weld['Tmax_C'], label='Tmax during welding')
plt.scatter(inter['time_s'], inter['TinterPass_C'],
            color='red', label='Inter-pass temperature')
plt.axhline(1450, linestyle='--', label='Solidus 1450C')
plt.xlabel('Time [s]'); plt.ylabel('Temperature [C]')
plt.legend(); plt.show()
```

---

## Key Physics Insights

### Cumulative thermal history
Repeated passes at rising z-height build up thermal strain in lower layers.
Each layer deposits new heat onto already-stressed material. After 26 layers,
the base plate experiences significantly larger residual stresses than any single pass.

### Inter-pass temperature control
The 60s inter-pass wait allows the deposit to cool before the next layer.
Too short: excessive heat buildup raises inter-pass T above 300C, reducing
yield strength and increasing distortion. Too long: cold start requires more
energy to remelt the previous layer top for good fusion.

### Thermal strain accumulation
Each heating and cooling cycle adds `epsTh += alpha * dT`. After 26 layers
the cumulative strain in the lower layers is much larger than in the upper layers
because lower layers have experienced more thermal cycles. This gradient drives
the characteristic bow distortion of WAAM walls.

### Element activation accuracy
Multiplying the Goldak source by `psiActive` is essential. Without it, the
Gaussian tail deposits energy into quiet elements with `Cp_eff = 1e-4 * Cp_real`,
causing temperature spikes of 10^7 C. The masking ensures the source cannot heat material that does not yet exist.

---

## Common Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| `Tmax > 5000C` | Goldak source hits quiet elements (Cp near zero) | Ensure `Tsource *= psiActive` is applied |
| `Tmax = 137000C` | Bulk layer activation before property update | Use progressive x-based activation inside weld loop |
| `Tmax = 40000C` | `TdepInit = Tliq` + Goldak source = double heating | Set `TdepInit = Tamb`; let source do the heating |
| Blue ghost layers in ParaView | Inactive nodes rendered at TempC=25C | Apply Threshold: Activation > 0.5 |
| `movemesh3` parser error | `sin/cos` in transfo on Windows v4.15 | Use only linear `scalar*variable` in transfo |
| `fespace` parse error after `movemesh3` | `pi` built-in in transfo | Declare `real piVal=3.14159...` and use that |
| F=0 in mechanical solve | Wrong boundary labels from `buildlayers` | Use `cube()` + `movemesh3`; labels 1=bottom, 2=top |

---

## Extending the Model

| Extension | What to change |
|---|---|
| Bidirectional deposition | Alternate torch direction each layer (x=0->L then L->0) |
| Multi-bead wall (wider) | Add second parallel bead pass per layer |
| Different alloy (Al, Ti) | Update `kSteel`, `CpSteel`, `rhoS`, `Tsol`, `Tliq`, `nuS` |
| Distortion measurement | Sample `dispM` at specific nodes vs time; compare to experiment |
| Residual stress validation | Export `sigM` at thermocouple positions; compare to XRD data |
| Interpass temperature control | Add adaptive `tWait` to target `Tmax < 300C` before next layer |
| Phase transformation | Add austenite/martensite fraction field coupled to temperature |
| Finer mesh near deposit | Use graded `movemesh3` with larger Nz in deposit zone |

---

## Citation

```bibtex
@software{mishra2026waam,
  author       = {Mishra, Akshansh},
  title        = {WAAM-MultiLayer-Simulation},
  year         = {2026},
  publisher    = {Zenodo},
  doi          = {10.5281/zenodo.20753459},
  url          = {https://doi.org/10.5281/zenodo.20753459}
}
```

---

## Author

**akshansh11**
GitHub: https://github.com/akshansh11

---

## License

<p>
<a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">
<img alt="Creative Commons Licence" style="border-width:0"
  src="https://i.creativecommons.org/l/by-nc/4.0/88x31.png"/>
</a>
<br/>
This work is licensed under a
<a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">
Creative Commons Attribution-NonCommercial 4.0 International License</a>.
</p>

You are free to:
- **Share** — copy and redistribute in any medium or format
- **Adapt** — remix, transform, and build upon the material

Under the following terms:
- **Attribution** — give appropriate credit and link to this repository
- **NonCommercial** — not for commercial use without permission

Copyright 2026. All rights reserved for commercial use.
