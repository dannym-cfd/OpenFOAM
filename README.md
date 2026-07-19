# CFD Portfolio

Two CFD case studies: race car external aero, supersonic missile aero. Each
includes a technical writeup and the OpenFOAM case files.

- [Race Car External Aero — GPU-Accelerated RANS/PIMPLE](racecarAero/)
- [Supersonic Missile Aerodynamics — Mach 3.8115](superSonic/)

## Race Car External Aero

Half-car symmetry-plane simulation at 50 m/s: vortex generators, underbody
diffuser, engine intake, front splitter, rear spoiler. Steady RANS
(`simpleFoam`) precursor followed by transient PIMPLE. GPU pressure solve
via a custom AmgX/CUDA-aware OpenMPI build (`amgx4Foam`), 12 cores.
Result: **Cd 0.278, Cl -0.180**. Details: [case study](racecarAero/).

![Streamline visualization over the race car, colored by surface pressure, on a black background](assets/img/racecarAero-streamline-render.jpg)

## Supersonic Missile Aerodynamics

Two-stage boosted missile at Mach 3.8115, 15.77 km altitude, from a
point-mass ascent trajectory model. Two-mesh coarse-to-fine workflow
(175k-cell coarse case mapped onto a 1.23M-cell fine case) for the
compressible `sonicFoam` solve. Result: **Cd 0.0310**, near-zero
lift/side-force/moments as expected for an axisymmetric body at zero angle
of attack. Details: [case study](superSonic/).

![Schlieren-style rendering of the missile in supersonic flight showing the bow shock cone](assets/img/missile-shock-render.jpg)

## Workstation

- CPU: AMD Ryzen 9 7900X
- RAM: 64 GB DDR5 @ 6200 MHz
- GPU: NVIDIA RTX 3080
