# Top Stage — Wing-Sweep Study at Mach 0.70

The glide (second) stage of the [supersonic missile](../) on its own, with
the variable-sweep wings set at five angles: 0°, 20°, 40°, 60° and 86°. All
cases share one freestream (Mach 0.70, sea-level ISA, 0° AoA) and the same
reference values (body frontal area, length 0.966 m, CG at z = -0.519 m), so
their force coefficients compare directly.

[`make_sweep_cases.py`](make_sweep_cases.py) builds the case matrix from the
per-angle STLs and a template case: freestream and reference values,
snappyHexMesh levels per surface (wings one level finer, plus refinement
boxes around each wing root where it passes through the skin), and the
surfaceFeatureExtract intersection that gives snapping the wing/skin edge.

## Cases

| Case | Solver | Turbulence | Status |
|---|---|---|---|
| `topstage00_M0.70` … `topstage60_M0.70` | rhoSimpleFoam | kOmegaSST | set up, not yet solved |
| `topstage86_M0.70` | rhoSimpleFoam | kOmegaSST | Cd 0.509, Cl 2.58 at iteration 6272 |
| `topstage86_M0.70_LTS` | rhoPimpleFoam, local time stepping | realizableKE | Cd 0.509, Cl 1.41 at iteration 15175 |

The 86° case was rerun with LTS `rhoPimpleFoam` and `realizableKE` (to match
the full-missile case's wall treatment); the original kOmegaSST setup is kept
in `backup_kOmegaSST_20261013/`. Force, residual, y+ and continuity histories
are in each solved case's `postProcessing/`.

Each case folder holds `Allmesh`/`Allclean`, `0.orig`, `constant`
(thermophysics, turbulence, surface STLs) and `system`. Meshes and
time directories are not included — run `Allmesh` to regenerate the mesh.

[← Supersonic missile](../README.md) · [All projects](../../README.md)
