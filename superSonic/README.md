# Supersonic Missile Aerodynamics — Mach 3.8115

Two-stage boosted missile (nose, fuselage, body fins, booster fins) at a
flight condition from a point-mass ascent trajectory model, using a
two-mesh coarse-to-fine workflow for a 1.2M-cell compressible solve.

| Mach | Altitude | Cd | Fine mesh | Solver |
|---|---|---|---|---|
| 3.8115 | 15.77 km | 0.0310 | 1.23M cells | sonicFoam |

## Problem & Goal

Compressible aero of a two-stage boosted missile at a supersonic condition:
converged drag coefficient, and confirmation that shock structure and
near-zero lift/side-force/moments (axisymmetric body, zero AoA) are
physical. Flight condition taken from a point-mass ascent trajectory model
(velocity, Mach, altitude, thrust, drag, stage, propellant mass, stability
margin) at ~19 s into flight, mid first-stage burn.

## Methodology

### Flow conditions

- Mach 3.8115, 1124.9 m/s, 15.77 km altitude (ISA stratosphere).
- p∞ = 10,663 Pa, T∞ = 216.65 K, ρ∞ = 0.1714 kg/m³ (ideal gas law, within
  rounding).
- Inlet: `supersonicFreestream`. Outlet/sides: `inletOutlet` (U/T),
  `waveTransmissive` (p) — non-reflecting outflow.
- Thermophysical: `hePsiThermo` / `perfectGas`, air (Cp 1005, μ 1.8×10⁻⁵,
  Pr 0.7).
- Turbulence: RAS Launder-Sharma k-ε (low-Re, wall-resolved).

### Two-mesh strategy

Coarse mesh run first, mapped onto the fine mesh, avoiding a cold-start
shock-formation transient on the expensive mesh:

| | Coarse mesh | Fine mesh |
|---|---|---|
| Cells | 175,166 | 1,232,688 |
| Faces | 558,029 | 3,856,056 |
| Fin refinement level | 6 | 8 |
| Timestep | 4×10⁻⁷ s | 9×10⁻⁸ s |
| End time | 0.03 s | 0.005 s |

A `coarsemap` script reconstructs the coarse case's latest timestep and runs
`mapFields -consistent` onto the fine mesh. 611,508 of 1,232,688 fine-mesh
cells sit at the finest level, concentrated at the fins. Boundary layers
reached 55–86% of target thickness by patch; mesh passed all quality checks,
zero illegal faces.

### Solver strategy

`sonicFoam`, Euler time stepping, bounded/TVD schemes for shocks/expansion
fans. Decomposition: `scotch` (minimizes inter-processor boundaries on
complex geometry), 12 cores, single workstation. Fixed 9×10⁻⁸ s timestep
kept mean Courant ≈0.00028 (max 0.364).

## Engineering Challenges

- **A GPU linear-solver stack was built, then not shipped.** Built from
  source: CUDA-aware OpenMPI, PETSc (standard + CUDA-MPI), `petsc4Foam`,
  `foam2csr`, `amgx4Foam`, plus patches for known AmgX integration bugs.
  Production runs use OpenFOAM's native CPU `DILUPBiCGStab` — no `libs`
  entry for AmgX in the shipped `controlDict`. GPU path hit integration
  friction against this OpenFOAM version's API; CPU path shipped instead.
  (Same GPU-solver infrastructure is integrated on the race car project —
  see that case study.)
- **Solver choice pivoted mid-project.** Early setup used `rhoPimpleFoam`
  (transient PIMPLE, adjustable local timestepping). Final case moved to
  `sonicFoam`, fixed small timestep, explicit PISO — simpler and more
  robust for genuinely supersonic flow.
- **Mesh-quality issues tracked to specific cells.** An earlier pass had 9
  illegal faces at identifiable cell IDs; final pass: 0 illegal faces.

## Results

At t = 0.00492102 s, ρ∞ = 0.1714, l_ref = 1.974 m, A_ref = 0.0791 m²
(body diameter ≈ 0.317 m):

| Coefficient | Value |
|---|---|
| Cd (total) | 0.0310 |
| Cd, forward half | 0.0155 |
| Cd, rear half | 0.0155 |
| Cl, Cs, Cm (pitch/roll/yaw) | ≈ 0 (as expected) |

Drag splits evenly forward/rear; lift/side-force/moment ≈ 0, as expected for
an axisymmetric body at zero AoA. Cd holds at 0.0309–0.0310 (±0.0002) over
the final ~0.0008 s.

![Schlieren-style rendering of the missile in supersonic flight, showing the bow shock cone forming around the nose and body, and a turbulent wake shed from the tail fins.](../assets/img/missile-shock-render.jpg)

*Density/pressure-gradient visualization at Mach 3.8: bow shock off the nose
and body, turbulent wake from the tail fins.*

![Streamline visualization around the missile body showing flow deflection around the nose and fins.](../assets/img/missile-streamlines.jpg)

*Streamlines around the nose, body, and fin geometry.*

> Flight condition (Mach/altitude) was read directly from a time-resolved
> ascent trajectory model, not picked as a round test number — the result
> reflects a specific point in a modeled flight profile.

[← Back to all projects](../README.md) · [Race Car Aero](../racecarAero/)
