# Supersonic Missile Aerodynamics — Mach 3.8115

A two-stage boosted missile (nose, fuselage, body fins, booster fins)
simulated at a flight condition derived from a custom point-mass ascent
trajectory model, using a two-mesh coarse-to-fine workflow to make a
1.2M-cell compressible solve tractable.

| Mach | Altitude | Cd | Fine mesh | Solver |
|---|---|---|---|---|
| 3.8115 | 15.77 km | 0.0310 | 1.23M cells | sonicFoam |

## Problem & Goal

Compressible external aerodynamics of a two-stage boosted missile at a
supersonic flight condition, targeting a converged drag coefficient and
confirmation that the flow field (shock structure, near-zero
lift/side-force/moments for an axisymmetric body at zero angle of attack)
behaved physically. The flight condition itself wasn't picked arbitrarily —
it was read off a custom-built point-mass ascent trajectory model
(time-resolved velocity, Mach, altitude, thrust, drag, stage, propellant
mass, and stability margin), at roughly 19 seconds into flight, mid-way
through first-stage burn.

## Methodology

### Flow conditions

- Mach 3.8115, freestream velocity 1124.9 m/s, at 15.77 km altitude (ISA
  stratosphere).
- p∞ = 10,663 Pa, T∞ = 216.65 K, ρ∞ = 0.1714 kg/m³ — internally consistent
  via the ideal gas law to within rounding.
- Inlet: `supersonicFreestream` boundary condition. Outlet and side
  boundaries: `inletOutlet` for U/T and `waveTransmissive` for p, a
  non-reflecting outflow treatment chosen specifically to avoid
  pressure-wave reflection back into the domain at supersonic speed.
- Thermophysical model: `hePsiThermo` / `perfectGas`, standard air
  properties (Cp 1005, μ 1.8×10⁻⁵, Pr 0.7).
- Turbulence: RAS Launder-Sharma k-ε, a low-Reynolds-number variant
  appropriate for wall-resolved boundary layers rather than wall functions.

### Two-mesh strategy

Rather than starting the expensive fine mesh from a uniform freestream and
burning through the initial shock-formation transient at full cost, the
workflow runs a coarse mesh first, then maps its result onto the fine mesh:

| | Coarse mesh | Fine mesh |
|---|---|---|
| Cells | 175,166 | 1,232,688 |
| Faces | 558,029 | 3,856,056 |
| Fin refinement level | 6 | 8 |
| Timestep | 4×10⁻⁷ s | 9×10⁻⁸ s |
| End time | 0.03 s | 0.005 s |

A custom `coarsemap` script reconstructs the coarse case's latest timestep
and runs `mapFields -consistent` onto the fine mesh, so the fine mesh
inherits an already-established flow field instead of starting cold. Over
half the fine mesh's cells (611,508 of 1,232,688) sit at the finest
refinement level, concentrated at the fins where the flow gradients are
sharpest. Boundary layers reached 55–86% of target thickness depending on
patch (fuselage lowest, body fins highest) — not fully at target
everywhere, but the mesh still passed all quality checks with zero illegal
faces in the final pass.

### Solver strategy

`sonicFoam` (PISO-based compressible solver), Euler time stepping,
bounded/TVD convective schemes appropriate for strong gradients near shocks
and expansion fans. Decomposition used the `scotch` method specifically
because it minimizes inter-processor boundaries automatically on complex
snappyHexMesh-generated geometry, run across 12 cores on a single
workstation. The fine-mesh run's fixed 9×10⁻⁸ s timestep, sized for
stability at Mach 3.8 through the finest cells, kept the mean Courant number
down around 0.00028 (max 0.364) at the end of the run.

## Engineering Challenges

- **A GPU linear-solver stack was built, then not shipped.** Significant
  infrastructure was built from source in a dedicated build tree: a
  CUDA-aware OpenMPI, PETSc (both standard and CUDA-MPI variants),
  `petsc4Foam`, `foam2csr`, and `amgx4Foam`, including patch scripts for
  known AmgX integration bugs and single-rank validation tests. Despite
  that effort, the production runs use OpenFOAM's native CPU
  `DILUPBiCGStab` solver — no `libs` entry for AmgX appears in the shipped
  `controlDict`. The GPU path hit integration friction against this
  OpenFOAM version's API and the robust, already-working CPU path was
  shipped instead of blocking the result on it. (This is the same
  underlying GPU-solver infrastructure that *did* get fully integrated on
  the race car project — see that case study — so the effort wasn't
  wasted, just not applicable to this solver in time.)
- **Solver choice pivoted mid-project.** An early setup used
  `rhoPimpleFoam` (transient PIMPLE with adjustable local timestepping).
  The final case moved to `sonicFoam` with a fixed, small timestep and
  explicit PISO — a robustness/simplicity decision appropriate for a
  genuinely supersonic (not merely transonic) flow regime, where the
  adjustable local-timestepping approach added complexity without a clear
  benefit.
- **Mesh-quality issues were tracked down to specific cells, not just
  avoided.** An earlier meshing pass produced 9 illegal faces at
  identifiable cell IDs; the final mesh generation pass resolved this
  cleanly (0 illegal faces), rather than being patched over after the fact.

## Results

Fine-mesh converged coefficients at t = 0.00492102 s (near the 0.005 s end
time), reference values ρ∞ = 0.1714, l_ref = 1.974 m (body length),
A_ref = 0.0791 m² (frontal area, body diameter ≈ 0.317 m):

| Coefficient | Value |
|---|---|
| Cd (total) | 0.0310 |
| Cd, forward half | 0.0155 |
| Cd, rear half | 0.0155 |
| Cl, Cs, Cm (pitch/roll/yaw) | ≈ 0 (as expected) |

Drag splits almost exactly evenly between the forward and rear halves of the
body, and lift/side-force/moment coefficients are all effectively zero — the
expected result for an axisymmetric body at zero angle of attack, and a
useful sanity check that the simulation setup (mesh symmetry, BCs) isn't
introducing spurious asymmetric forces. The time history over the final
~0.0008 s of simulated time shows Cd holding steady at 0.0309–0.0310 with
only small (±0.0002) oscillation — a well-converged estimate, not a single
noisy snapshot.

![Schlieren-style rendering of the missile in supersonic flight, showing the bow shock cone forming around the nose and body, and a turbulent wake shed from the tail fins.](../assets/img/missile-shock-render.jpg)

*Density/pressure-gradient visualization of the Mach 3.8 flow field, showing
the bow shock forming off the nose and body, and turbulent wake structure
shed from the tail fins.*

![Streamline visualization around the missile body showing flow deflection around the nose and fins.](../assets/img/missile-streamlines.jpg)

*Streamline visualization of the flow field around the nose, body, and fin
geometry.*

> **On the flight condition:** the Mach/altitude point used for this CFD
> snapshot was read directly from a time-resolved ascent trajectory model
> (velocity, Mach, altitude, thrust, drag, stage, propellant mass, stability
> margin) rather than picked as a round test number — the CFD result
> reflects a specific, physically modeled moment in a realistic flight
> profile.

[← Back to all projects](../README.md) · [Race Car Aero](../racecarAero/) · [Subaru Outback](../subaruOutback/)
