# CFD Portfolio

Three CFD case studies (race car external aero, supersonic missile aero,
Subaru Outback aero), each with full technical writeups and the underlying
OpenFOAM case files.

- [Race Car External Aero — GPU-Accelerated RANS/PIMPLE](racecarAero/)
- [Supersonic Missile Aerodynamics — Mach 3.8115](superSonic/)
- [Subaru Outback (2022) — Automotive External Aero](subaruOutback/)

## Race Car External Aero

Half-car symmetry-plane simulation of a full aero package — vortex
generators, underbody diffuser, engine intake, front splitter, rear spoiler
— at 50 m/s, using a steady RANS (`simpleFoam`) precursor followed by a
transient PIMPLE run once the wake proved genuinely unsteady. Pressure was
solved on GPU via a custom-built AmgX/CUDA-aware OpenMPI integration
(`amgx4Foam`) across 12 cores. Converged result: **Cd 0.278, Cl -0.180**
(net downforce). Full writeup covers the snappyHexMesh refinement strategy,
several real mesh-crash debugging stories, and the y+ validation — see the
[case study](racecarAero/).

![Streamline visualization over the race car, colored by surface pressure, on a black background](assets/img/racecarAero-streamline-render.jpg)

## Supersonic Missile Aerodynamics

A two-stage boosted missile simulated at Mach 3.8115, 15.77 km altitude — a
flight condition read directly off a custom-built point-mass ascent
trajectory model rather than picked arbitrarily. Uses a two-mesh
coarse-to-fine field-mapping workflow (175k-cell coarse case feeding a
1.23M-cell fine case via `mapFields`) to make the compressible `sonicFoam`
solve tractable. Converged result: **Cd 0.0310**, split almost exactly
evenly between the forward and rear halves of the body, with near-zero
lift/side-force/moments as expected for an axisymmetric body at zero angle
of attack. Full writeup includes an honest account of a GPU linear-solver
stack (AmgX/PETSc/foam2csr) that was built from source but ultimately not
shipped in production — see the [case study](superSonic/).

![Schlieren-style rendering of the missile in supersonic flight showing the bow shock cone](assets/img/missile-shock-render.jpg)

## Workstation

All cases were meshed and solved locally:

- CPU: AMD Ryzen 9 7900X
- RAM: 64 GB DDR5 @ 6200 MHz
- GPU: NVIDIA RTX 3080 (used for the AmgX-accelerated pressure solves)
