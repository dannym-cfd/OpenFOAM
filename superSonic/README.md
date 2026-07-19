# Supersonic Missile Aerodynamics — Mach 3.8115

Two-stage boosted missile (nose, fuselage, body fins, booster fins) at a
flight condition from a point-mass ascent trajectory model, using a
two-mesh coarse-to-fine workflow for a 1.2M-cell compressible solve.

| Mach | Altitude | Cd | Fine mesh | Solver |
|---|---|---|---|---|
| 3.8115 | 15.77 km | 0.0310 | 1.23M cells | sonicFoam |

## Goal

The purpose of this case is to get a trustworthy drag coefficient for a
real, physically-grounded flight condition, and confirm the flow physics
around the vehicle behave the way they should for a supersonic, axisymmetric
body — not just to run a CFD solve and report whatever number comes out.
Concretely, that means two things: a converged Cd at a specific point in the
missile's flight, and a sanity check that the shock structure looks physical
and that lift/side-force/moments all come out near-zero, which is exactly
what should happen for an axisymmetric body at zero angle of attack. If
those secondary checks came out wrong, the Cd number wouldn't be trustworthy
regardless of how "converged" it looked.

The vehicle itself is a two-stage design: **Stage 1** is a solid rocket
motor sustainer providing powered boost; **Stage 2** is an unpropelled glide
stage with variable-sweep wings that deploy once the booster separates, with
sweep angle set by the current flight state rather than fixed. The specific
flight condition simulated — Mach 3.8115 at 15.77 km altitude — isn't a
round test number. It's a single point pulled from a custom flight-mechanics
model (below), sampled at ~19 s into flight, mid-way through first-stage
burn.

## Process

Getting from "we want a Cd at this flight condition" to an actual converged
CFD result took four stages: defining the physical vehicle (airframe/CAD),
figuring out what flight condition was actually worth simulating (the MATLAB
flight-mechanics tool), setting up and solving the CFD case itself, and then
verifying that solve actually converged before trusting the result. Each is
covered below, along with the real engineering dead ends hit along the way.

### Airframe & Avionics

Before any CFD or trajectory math happens, the vehicle needs to actually
exist as a defined, dimensioned object — otherwise "Mach 3.8115" is just a
number with nothing physical behind it. This section covers the CAD
assembly, the exact geometry (airfoils, pivot locations, nose profile), and
the avionics that would fly on a real build.

Body diameter 98 mm, total (glider-stage) mass 9180.43 g. Wing pivot at
-494.5 mm from nose tip, control fins centered at -930.5 mm from nose tip.

| Component | Airfoil | Chord | Span |
|---|---|---|---|
| Main wings (deployable) | NASA SC(2)-0710 | 85 mm | 300 mm |
| Control fins | NACA 0008 | 50 mm | 100 mm |

Wing deployment: fully retracted → deployed, max actuator rate 15°/s,
starting from a Mach 2 deployment condition.

**Nose cone**: Von Kármán ogive (Haack series, C = 0 — the minimum-drag
profile for a given length/base diameter under supersonic slender-body
theory). Base radius 49 mm, fineness ratio L/D = 5 (L = 490 mm, adjustable
parameter). Profile: θ = acos(1 − 2x/L), y = (R/√π)·√(θ − sin(2θ)/2 + C·sin³θ).

CAD assembly (Fusion 360) of the full vehicle: nose, electronics bay,
variable-sweep wing assembly, and booster.

![Full two-stage stack, side view, showing nose, electronics bay with antenna fairings, swept wings, and booster with tail fins](../assets/img/missile-cad-fullstack.png)

*Full stack, side view: nose, electronics bay (yellow band + antenna
fairings), wings, booster body and tail fins.*

![Second stage (glider) alone, front view, with wings deployed and no booster](../assets/img/missile-cad-topstage.png)

*Second stage (glider) alone: nose, electronics bay, deployed wings, and the
tail control-fin assembly — no booster or booster fins attached.*

![Exploded view of the electronics bay module showing individual component blocks between the nose and booster](../assets/img/missile-cad-elecbay.png)

*Electronics bay, exploded view. Sensor suite: LoRaWAN GPS tracker,
MS5611-01BA03 barometric pressure sensor, Bosch BMI085 IMU (QFN), u-blox
NEO-M9N-00B GNSS module (SMT).*

![Close-up of the wing root showing the two linear actuators driving wing sweep](../assets/img/missile-cad-actuators.png)

*Wing-sweep actuator mechanism at the wing root — two linear actuators per
side driving the sweep motion `missileFlightApp` models as `sweep_deg`.*

### Flight Profile — missileFlightApp

With the vehicle defined, the next question is: at what point in an actual
flight is it worth spending CFD compute? A missile's Mach number and
altitude change continuously through boost and glide, so picking a
condition to simulate needs an actual flight-mechanics model behind it —
otherwise "Mach 3.8" is arbitrary. `missileFlightApp` is a custom MATLAB
tool built to answer exactly that: a point-mass trajectory model covering
the full flight profile (loft trajectory, engine/sustainer burn, staging,
and the unpropelled glide phase), computing:

- Maximum range for a given impact speed
- Loft trajectory (altitude/velocity/Mach vs. time)
- Staging events (sustainer burnout, second-stage separation)
- Variable-sweep wing angle vs. flight state on the glide stage
- Fin deflection angle

The Mach 3.8115 / 15.77 km CFD condition used throughout this case is a
single point pulled directly from this model's output.

#### App layout

Five tabs: Flight Path (trajectory, velocity/ground track, forces &
atmospheric density), Aircraft Characteristics (CG migration, wing sweep,
fin trim deflection), Engine (motor design + thrust/propellant-burn plots),
Geometry (2D top-view animation), Data Table (full per-timestep output,
right-click export to CSV).

![Flight Path tab showing trajectory (max altitude 31.2 km, distance 224.0 km), velocity/ground track, and active forces & atmospheric density plots](../assets/img/missileFlightApp-flightpath.png)

*Flight Path tab: trajectory, Mach/ground-track history, and force/density
history for the default launch parameters.*

![Aircraft Characteristics tab showing dynamic CG migration, wing sweep vs flight path angle, and active flight trim deflection plots](../assets/img/missileFlightApp-aircraft-characteristics.png)

*Aircraft Characteristics tab: CG migration, wing sweep angle vs. flight
path angle, and fin trim deflection over the same flight.*

#### Integration and CFD feedback loop

A flight-mechanics model is only as good as the aerodynamic assumptions
feeding it, so this isn't a one-way pipeline — CFD results feed back into
the trajectory model to correct its analytical drag estimate. Trajectory
integration itself is RK4, replacing an earlier explicit-Euler scheme —
Euler's numerical damping was under-predicting peak Mach (3.81 vs. RK4's
4.3) and range during the high-thrust boost phase. `readLatestForceCoeffs.m`
scans the OpenFOAM case's `postProcessing/forceCoeffs/` for the most
recently written result and computes a calibration factor
(`Cd_CFD / Cd_analytical`) applied to stage-1 drag. Since the CFD geometry
covers the whole boosted stack (nose/fuselage/body fins/booster fins) with
no separate glide-fin surface, the factor is applied only to those terms —
an earlier version incorrectly scaled tail-fin drag with a factor never
validated against it. Cl/CmPitch calibration is also read from CFD but
skipped when the analytical baseline is ~0 (AoA=0 on a symmetric body),
since the ratio is undefined there.

#### Motor model

The boost-phase trajectory (and therefore the flight condition itself)
depends on how the solid rocket motor is sized, so the motor needed its own
parametric model rather than a fixed assumed thrust curve. Solid rocket
motor sized from a fixed CAD envelope (88mm OD × 1m length): wall thickness
and material (Aluminum/Steel/Titanium/CarbonFiber/Fiberglass) set casing
mass and remaining bore volume; propellant mass falls out of that bore
volume × propellant density. Propellant type (APCP, double-base, black
powder, HTPB/AP/Al) and nozzle expansion ratio drive an effective Isp via a
nozzle-efficiency curve. This is a parametric approximation, not a real
internal-ballistics model — no grain burn-area geometry, no
chamber-pressure/burn-rate coupling.

![Engine tab showing Motor Design panel (APCP propellant, aluminum wall, 20:1 nozzle expansion ratio, 3mm wall thickness, 8000N boost thrust) and Engine Thrust / Propellant Burn plots over the 20s burn](../assets/img/missileFlightApp-engine.png)

*Engine tab: motor geometry/propellant inputs on the left (bore ID 82.0mm,
9.242kg propellant, 2.163kg casing, 227.1s effective Isp for this
configuration), thrust profile (8000N boost dropping to a lower sustain
level) and propellant mass depletion over the 20.5s burn on the right.*

#### Geometry rendering and debugging

The app's 2D geometry view exists to visually sanity-check that the
trajectory model's staging, wing sweep, and CG behavior are actually doing
what the underlying physics says they're doing — and building it surfaced
some real bugs along the way. Nose, fuselage, and booster fins render from
real STL outlines (sliced at the Y=0 plane, top-view, so a laterally-swept
wing is actually visible rather than edge-on). The wing itself is a
synthetic constant-chord rectangle driven by the same sweep formulas as the
aero model — real-STL wing rotation was tried and reverted after it
produced degenerate, flying-off shapes at large sweep angles, traced to
rotating about a pivot that didn't match the STL's actual coordinate frame.

Two behaviors that looked like bugs but weren't: the top-stage CG appeared
to toggle between two fixed points instead of moving continuously with wing
sweep, which turned out to be a real, continuous ~5mm shift on a ~965mm
body — correct physics, just below the plot's visible resolution. Booster
separation went through several corrections based on the real staging
behavior (booster body and its fins jettison together as one unit at
stage separation, not fins alone) before the drawing and the mass/CG
physics in `simulateFlight.m` agreed.

![Geometry tab, stage 1, showing the boosted stack top-view with stowed wing (90° sweep), booster fins, and engine plume at Mach 1.20](../assets/img/missileFlightApp-geometry-stage1.png)

*Geometry tab, stage 1: full boosted stack, wing stowed (90° sweep) inside
the fuselage, booster fins attached, engine plume rendered from
instantaneous thrust.*

![Geometry tab, stage 2, showing the glider alone with the wing swept out to 22.8° and the tail control fin deflected -3.9° at Mach 0.98](../assets/img/missileFlightApp-geometry-stage2.png)

*Geometry tab, stage 2: booster (with its fins) jettisoned, wing swept out
to 22.8°, tail fin trimmed to -3.9° at Mach 0.98.*

### CFD Setup

With a specific flight condition chosen, the CFD case needs flow physics and
a mesh that can actually resolve a Mach 3.8 shock structure without either
being wrong (too coarse) or impossibly expensive (too fine everywhere at
once) — which is why this uses a two-mesh strategy rather than a single
mesh.

**Flow conditions:**

- Mach 3.8115, 1124.9 m/s, 15.77 km altitude (ISA stratosphere).
- p∞ = 10,663 Pa, T∞ = 216.65 K, ρ∞ = 0.1714 kg/m³ (ideal gas law, within
  rounding).
- Inlet: `supersonicFreestream`. Outlet/sides: `inletOutlet` (U/T),
  `waveTransmissive` (p) — non-reflecting outflow.
- Thermophysical: `hePsiThermo` / `perfectGas`, air (Cp 1005, μ 1.8×10⁻⁵,
  Pr 0.7).
- Turbulence: RAS Launder-Sharma k-ε (low-Re, wall-resolved).

**Two-mesh strategy:** rather than starting the expensive fine mesh from a
cold, uniform freestream and burning compute on the initial shock-formation
transient, a coarse mesh is run first and mapped onto the fine mesh:

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

Mesh generation (snappyHexMesh) iteration counts and final checkMesh-equivalent
result, both meshes clean (all checks 0, "Finished meshing without any errors"):

| | Coarse mesh | Fine mesh |
|---|---|---|
| Castellation refinement iterations | 7 | 12 |
| Snapping iterations | 41 | 41 |
| Layer addition iterations | 27 | 11 |

### Solver Strategy

Meshing alone doesn't guarantee a usable solve — the solver, scheme choice,
and decomposition all had to be matched to a genuinely supersonic, shock-
containing flow rather than a generic RANS setup. `sonicFoam`, Euler time
stepping, bounded/TVD schemes for shocks/expansion fans. Decomposition:
`scotch` (minimizes inter-processor boundaries on complex geometry), 12
cores, single workstation. Fixed 9×10⁻⁸ s timestep kept mean Courant
≈0.00028 (max 0.364).

### Convergence Verification

A converged-looking Cd number is only trustworthy if the mesh that produced
it actually resolved the geometry properly and the solve itself settled
rather than just stopped. This section is the evidence for both, covering
coarse vs. fine mesh side by side.

#### Cells per refinement level

<table>
<tr><th>Coarse mesh</th><th>Fine mesh</th></tr>
<tr>
<td><img src="../assets/img/missile-coarse-cells-per-level.png" width="100%"></td>
<td><img src="../assets/img/missile-fine-cells-per-level.png" width="100%"></td>
</tr>
</table>

*Coarse mesh tops out at refinement level 7; fine mesh reaches level 8,
which dominates at 611,508 of 1,232,688 total cells (~50%).*

| Metric | Coarse mesh | Fine mesh |
|---|---|---|
| Total cells | 175,166 | 1,232,688 |
| Max refinement level | 7 | 8 |
| Cells at max level | ~75,000 (~43%) | 611,508 (~50%) |

#### Layers per patch

<table>
<tr><th>Coarse mesh</th><th>Fine mesh</th></tr>
<tr>
<td><img src="../assets/img/missile-coarse-layers-per-patch.png" width="100%"></td>
<td><img src="../assets/img/missile-fine-layers-per-patch.png" width="100%"></td>
</tr>
</table>

*Coarse mesh only resolved layers on fuselage/bodyFins with low coverage
(~16% fuselage, ~0% bodyFins); fine mesh reaches full three-patch coverage
at 54–85%.*

| Patch | Coarse target layers | Coarse coverage | Fine target layers | Fine coverage |
|---|---|---|---|---|
| fuselage | 5 | ~16% | 20 | 54% |
| bodyFins | 3 | ~0% | 12 | 85% |
| boosterFins | — (not tracked) | — | 12 | 73% |

#### Solver results

Full-range solve logs:

<table>
<tr><th>Coarse mesh</th><th>Fine mesh</th></tr>
<tr>
<td><img src="../assets/img/missile-coarse-solver-timepacing.png" width="100%"></td>
<td><img src="../assets/img/missile-fine-solver-timepacing.png" width="100%"></td>
</tr>
</table>

*Time pacing: simulated time vs. wall-clock time, and cumulative average
solve rate.*

<table>
<tr><th>Coarse mesh</th><th>Fine mesh</th></tr>
<tr>
<td><img src="../assets/img/missile-coarse-residuals.png" width="100%"></td>
<td><img src="../assets/img/missile-fine-solver-initial.png" width="100%"></td>
</tr>
</table>

*Initial residuals per field.*

<table>
<tr><th>Coarse mesh</th><th>Fine mesh</th></tr>
<tr>
<td><img src="../assets/img/missile-coarse-solver-final.png" width="100%"></td>
<td><img src="../assets/img/missile-fine-solver-final.png" width="100%"></td>
</tr>
</table>

*Final (post-inner-iteration) residuals per field.*

<table>
<tr><th>Coarse mesh</th><th>Fine mesh</th></tr>
<tr>
<td><img src="../assets/img/missile-coarse-courant.png" width="100%"></td>
<td><img src="../assets/img/missile-fine-solver-courant.png" width="100%"></td>
</tr>
</table>

*Mean/max Courant number.*

| Metric | Coarse mesh | Fine mesh |
|---|---|---|
| Simulated time reached | ~0.024 s (of 0.03 s target) | ~0.0041 s (of 0.005 s target) |
| Wall-clock time | ~7,400 s (~2 h) | ~58,000 s (~16 h) |
| Mean Courant | ~1×10⁻³ | ≈0.00028 |
| Max Courant | ~0.35 | ≈0.364 |

Coarse mesh was also solved across two separate sessions on the same case
(run1/run2), compared directly:

<table>
<tr><th>Coarse mesh — initial residuals</th><th>Coarse mesh — final residuals</th></tr>
<tr>
<td><img src="../assets/img/missile-coarse-residuals-compare-initial.png" width="100%"></td>
<td><img src="../assets/img/missile-coarse-residuals-compare-final.png" width="100%"></td>
</tr>
</table>

*Coarse mesh, run1 (solid) vs. run2 (dashed): initial residuals (left),
final residuals (right).*

### Engineering Challenges

Not every part of the process worked on the first attempt — these are the
real dead ends and pivots encountered getting to the setup above, kept
because they're as informative as the final configuration itself.

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

The number this whole process was built to produce, and the physical
sanity checks that make it trustworthy rather than just a converged-looking
value.

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

Coarse-mesh field renders at the same t = 0.00492102 s:

![Pressure field on the coarse mesh, showing the bow shock envelope around the nose, fuselage, and fins](../assets/img/missile-coarse-pressure.jpg)

*Pressure field (coarse mesh): shock envelope visible around the nose, body,
and fins, freestream ≈1.5–4.6×10⁴ Pa scale.*

![Temperature field on the coarse mesh, showing shock heating along the body and wake](../assets/img/missile-coarse-temperature.jpg)

*Temperature field (coarse mesh): freestream ≈210 K, shock-heated air along
the body and wake reaching up to ≈970 K.*

![Velocity magnitude and density field on the coarse mesh around the missile body](../assets/img/missile-coarse-velocity.jpg)

*Velocity magnitude and density field (coarse mesh) around the body and
wake.*

> Flight condition (Mach/altitude) was read directly from a time-resolved
> ascent trajectory model, not picked as a round test number — the result
> reflects a specific point in a modeled flight profile.

[← Back to all projects](../README.md) · [Race Car Aero](../racecarAero/)
