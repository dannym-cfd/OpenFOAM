# Subaru Outback (2022) — Automotive External Aero

A custom-built vehicle model, taken through two design/geometry iterations
with measured drag improvement between them.

> **A note on this write-up:** this was an earlier project, and its live
> OpenFOAM case directory no longer exists (cleaned up before later projects
> began). Everything below about flow conditions, mesh, and solver choice is
> reconstructed from surviving Windows-side screenshots and files rather
> than confirmed against live dictionaries — it's presented as informed
> reconstruction, not verified fact, and marked accordingly. The force
> coefficient results themselves come directly from terminal output
> captured in those screenshots, so those numbers are solid.

| Cd (rev 1) | Cd (rev 2) | Cl (rev 2) | Geometry |
|---|---|---|---|
| 0.417 | 0.352 | -0.351 | Custom CAD |

## Problem & Goal

External aerodynamic simulation of a 2022 Subaru Outback XT, built as a
custom CAD model rather than sourced from an existing mesh library. The
model was built in Blender, then dimensionally checked against a real
trim-level parts/build-list spreadsheet to keep it faithful to an actual
production configuration rather than a generic stand-in shape. Separate
wheel, front-vent, and engine-block geometry were modeled, suggesting an
attempt at at least partial underhood cooling-flow detail, not just an
external shell.

## Methodology *(reconstructed — see caveat above)*

- Geometry: Blender-modeled base (`Subaru Outback 2022.blend`), exported to
  STL/OBJ, with separate front/rear wheel, front-vent, and engine-block
  surfaces. A second geometry revision was produced in Fusion 360 (naming
  convention: "obxt" = Outback XT, "Rev1" for the revised pass).
- Flow regime: low-speed/near-incompressible, consistent with a
  highway-speed automotive case (velocity contours in the ~40–44 m/s range)
  — nothing resembling the missile project's compressible regime.
- Turbulence/solver: no surviving dictionary confirms the exact model, but a
  RANS eddy-viscosity turbulence model and a steady-state solver (most
  likely `simpleFoam`) is the standard, near-universal choice for this class
  of problem and the best-supported inference from the visualized fields
  (steady-looking contours, no timestep-dependent transient artifacts).
- Mesh: not recoverable in detail. Visible stair-stepped contour boundaries
  in the wake region of the surviving renders are consistent with a
  snappyHexMesh-style background+refinement mesh, coarser than the level of
  detail used in the later race-car project.

## Results — Two Design Iterations

Two distinct sets of force coefficients survive, corresponding to the
Blender-based initial geometry and the Fusion-360-refined revision:

| | Rev 1 (Blender) | Rev 2 (Fusion 360 refined) |
|---|---|---|
| Cd | 0.417 | 0.352 |
| Cl | -0.651 | -0.351 |
| Cl, front axle | -0.425 | -0.344 |
| Cl, rear axle | -0.226 | -0.007 |
| Cm | -0.099 | -0.169 |

Drag dropped substantially (0.417 → 0.352) and total lift magnitude roughly
halved between the two geometry passes, with nearly all of the remaining
lift concentrated at the front axle in the revised geometry (rear-axle
contribution dropped to essentially zero). For context, production
Outback-class wagons typically report factory Cd figures in the 0.32–0.34
range, so 0.352 reads as a plausible, credible converged estimate, while
0.417 looks like an early/rough-geometry overestimate that the geometry
revision corrected.

> It isn't confirmed whether both runs used identical reference values
> (freestream velocity, reference area, reference length) — since no
> `forceCoeffsDict` survives from either run, some of the difference could
> in principle reflect changed reference parameters rather than purely the
> geometry revision. Presented here as the most likely explanation, not a
> certainty.

![Turbulent viscosity and pressure contour on the car's symmetry plane, showing a low-pressure wake region behind the hatch.](../assets/img/subaru-velocity-contour.jpg)

*Turbulent viscosity (nut) and pressure contour on the symmetry plane,
showing flow separation and a low-pressure wake behind the hatch.*

![Velocity magnitude and pressure contour along the car's profile, showing stagnation at the front fascia and an accelerated flow region over the roof.](../assets/img/subaru-pressure-contour.jpg)

*Velocity magnitude and pressure contour along the vehicle profile, showing
stagnation pressure at the front fascia and accelerated flow over the
roofline.*

[← Back to all projects](../README.md) · [Race Car Aero](../racecarAero/) · [Supersonic Missile](../superSonic/)
