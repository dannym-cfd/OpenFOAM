# Race Car External Aerodynamics — GPU-Accelerated RANS/PIMPLE

Half-car symmetry-plane simulation of a full aero package (vortex generators,
underbody diffuser, engine intake, sidepods) at 50 m/s, solved with OpenFOAM
v2512 and a GPU-accelerated pressure solve.

| Cd | Cl | Freestream | Turbulence | Cores |
|---|---|---|---|---|
| 0.278 | -0.180 | 50 m/s | k-ω SST | 12 |

## Problem & Goal

Full external aerodynamic simulation of a race car half-model (symmetry plane
at X = 0) at 50 m/s (~180 km/h), covering the complete aero package rather
than just a bare body: front/rear wheels with rotating-wall boundary
conditions, mirrors, a front splitter vent and side skirt, six roof-mounted
vortex generators, an engine intake flow path, an underbody diffuser strake,
and a rear spoiler spanning to the symmetry plane. The goal was a credible
steady-state drag/downforce baseline (Cd/Cl), with a transient follow-up once
the flow proved to have a genuinely unsteady wake rather than a numerical
artifact.

## Methodology

### Flow conditions & turbulence

- Freestream 50 m/s, k-ω SST turbulence model, ν = 1.5×10⁻⁵ m²/s.
- Inlet: fixed velocity (0, 50, 0) — coordinate convention Y = streamwise,
  Z = vertical, X = lateral/symmetry.
- Ground: moving-wall velocity matching freestream (correct relative-motion
  treatment for a stationary-frame car simulation).
- Wheels: `rotatingWallVelocity`, ω = 173.07 rad/s at 50 m/s, wheel radius
  0.2889 m.
- Engine intake (`engsim.stl`): modeled as a flow-rate outlet (not a wall) at
  0.392 m³/s, derived from a generic 3.0L twin-turbo V6 assumption (7500 rpm,
  1.2 bar boost, PR≈2.2, 95% volumetric efficiency). Extraction direction was
  verified by inspecting the solved flux field directly (100% positive flux
  = genuine extraction, not injection) rather than assumed from the BC alone.

### Mesh strategy

Built with snappyHexMesh across 12 parallel processes. Refinement levels
were tuned per-patch: body (4,5), wheels (5,6), mirror (6,6), and everything
else aerodynamically sensitive — front vent, side skirt, all six vortex
generators, engine intake blocker, diffuser fin, rear spoiler — at a uniform
(6,7) surface level, which was the proven-safe ceiling for surface-level
refinement on this geometry.

Level 8 refinement was needed locally at the vortex generators but caused
repeated snappyHexMesh crashes (segfaults in `hexRef8::setRefinement`, heap
corruption in `addLayers`) whenever applied as a surface-level refinement.
The fix was to apply level 8 only through small dedicated volumetric
refinement boxes around each VG (~0.0004–0.0009 m³ each) instead — never as
a surface level. An early version used one large shared box spanning all six
VGs (~0.027 m³, mostly empty roof space), which is why cell counts briefly
exploded; splitting it into six tight individual boxes fixed that.

A y+ check on an early-iteration field showed all patches healthy: the
ground averaged y+ ≈ 252 (down from 1594 before the ground got its own
boundary layers), and the vortex generators stayed well-behaved (avg 33–131,
max only 169–337, no near-wall instability).

### Solver strategy

Two-stage approach: a steady RANS precursor (`simpleFoam`,
SIMPLEC/`consistent yes`, relaxation on U/k/omega only, no p relaxation —
matching OpenFOAM's own motorBike tutorial reference settings rather than a
guessed configuration) run to residual plateau, followed by a transient
PIMPLE run (`pimpleFoam`, backward time scheme, 3 outer correctors) once
residual oscillation stopped decaying — the signature of a genuinely
unsteady wake rather than a numerics problem.

Pressure was solved on GPU via **amgx4Foam** (a direct AmgX integration, not
the PETSc AMGX wrapper), running over a custom-built CUDA-aware OpenMPI,
with an AGGREGATION AMG-preconditioned PCG solver. The transient run used
`maxCo 30` rather than the default-ish `maxCo 1`: with sub-millimeter
near-wall cells, `maxCo 1` Courant-limits the timestep down to ≈8×10⁻⁶ s,
which would have made 2 seconds of physical time take roughly two weeks of
wall-clock time. Letting bulk-flow cells set the pace instead cut that by
~30×.

## Engineering Challenges

- **Mismatched refinement levels between adjacent patches crash the
  mesher.** The engine intake blocker (`engSim`) sits spatially interleaved
  with the side skirt. Dropping its refinement to save cells (it's just a
  flat blocker) crashed snappyHexMesh at every lower level tried, because
  the level mismatch against its (6,7)-level neighbor produced bad
  transition cells. Fix: match neighboring patch levels exactly rather than
  optimizing each patch in isolation.
- **Layer-addition parameters have narrow safe ranges.** Dropping
  `minThickness` from 0.1 to 0.05 (chasing better layer coverage on the
  front vent) let near-degenerate, near-zero-height cells through, which
  blew up omega's wall function (ω ~ 1/y) to ~10⁵ from the very first
  iteration and crashed the solver immediately. Separately, setting `nGrow`
  to 1 (trying to close layer gaps at sharp edges) caused a catastrophic
  layer-coverage collapse across nearly the whole mesh, since it grows the
  do-not-extrude zone around every problem spot and this geometry has too
  many separate thin/sharp trouble spots for that not to cascade.
- **A plausible-sounding fix that didn't hold up under testing.** The
  hypothesis that volumetric refinement boxes would also improve
  boundary-layer coverage (not just castellation) was tested directly on
  five of the six vortex generators and found false — coverage stayed at
  13–19% regardless. The boxes were kept anyway for castellation
  consistency, but the assumption was discarded rather than carried forward
  unverified.
- **An early, badly-scaled BC produced a physically implausible result.**
  Before the engine intake velocity was corrected, an inflated value
  (30 m/s, roughly 3.2× too high) drove Cd to 0.702 and Cl to -0.898 — the
  oversized intake was disrupting the local flow field. Correcting the flow
  rate to the properly-derived 0.392 m³/s (and fixing the reference area
  used for coefficients) brought the result back in line with the converged
  baseline below.

## Results

Converged coefficients on the corrected-BC mesh (reference area 0.9314 m²:
9187.988 cm² half-model + 126 cm² wheel overhang):

| Coefficient | Value |
|---|---|
| Cd | 0.278 |
| Cl | -0.180 (net downforce) |

A large lateral force coefficient (Cs ≈ -0.807) also appeared in this result
and was investigated on the assumption it might be a bug — it wasn't. It's
an expected artifact of the half-model approach: a symmetric car's true side
force is zero by construction, and a half-model mesh with only one symmetry
plane can't compute that near-zero quantity meaningfully. Confirmed as
artifact, not chased further.

![Initial residuals vs simulated time for simpleFoam, showing U, p, k, and omega residual traces plateauing around t=200-1940 before a transient excursion near t=1950 that self-corrects.](../assets/img/racecarAero-residuals.jpg)

*Initial residuals vs. simulated time (simpleFoam). Residuals plateau rather
than fully converging from t≈200 onward — the signature that motivated the
switch to a transient PIMPLE run — with one large excursion around
t≈1941–1950 that self-corrects.*

![Velocity and pressure contour on the car's symmetry plane, showing stagnation pressure at the nose and a low-velocity wake region extending behind the car.](../assets/img/racecarAero-velocity-pressure.jpg)

*Velocity magnitude and pressure contours on the symmetry plane, from an
earlier mesh iteration (predating the final vortex-generator repositioning
and diffuser/spoiler geometry described above).*

> **On the GPU solve:** only 2 MPI ranks were ever formally validated as
> fully reliable with AmgX in isolated testing (4 ranks hung, 8+ crashed
> with pinned-memory errors) — but the production case ran reliably at 12
> ranks across many sessions. That gap between "formally validated" and
> "empirically working" is tracked as an open risk, not treated as resolved.

[← Back to all projects](../README.md) · [Supersonic Missile](../superSonic/) · [Subaru Outback](../subaruOutback/)
