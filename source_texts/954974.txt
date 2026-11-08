# Navier–Stokes Velocity Field Reconstruction

**Reproducible velocity fields. Independent residual checks. Interactive MATLAB visualization.**

This repository packages the strongest fully reproducible checkpoints selected from our constrained Navier–Stokes reconstruction project: **ST054-Q2**, the primary lower-L2 candidate, and **ST054-M3**, the lower-peak alternative. Both include velocity, pressure, a separately specified restricted force, frozen coefficients, and numerical evidence. No training is needed. A fresh source checkout restores checksum-verified assets from pinned research commits once; the complete downloadable bundle already includes them.

> **Research status:** the complete `1e-3` momentum target is **not yet met**. These are finite-window numerical candidates, not accepted exact NS solutions, recovered OpenAI fields, or a blow-up proof. This is an independent project, not an OpenAI repository.

## Start with the visualization

Download or clone this repository, open its root folder in MATLAB, and run:

```matlab
start_here                 % Full-field presentation preset, continuous time
start_here('core')         % Densely re-sampled central core + geometry metrics
start_here('full')         % Full interactive explorer without the preset
```

**First launch:** `start_here` calls `prepare_matlab`, which downloads only missing pinned MATLAB files and verifies every SHA-256 digest. An internet connection is needed for that initial source-checkout setup. The complete release ZIP is already initialized and can be used offline. Existing files with different bytes are not overwritten.

The native MATLAB viewers include a bottom time slider and playback, approximately 200 streamline seeds, vorticity surfaces, movable slices, pressure-force and full-residual displays, and PNG export. Q2 and M3 can be switched without rerunning a fit. The core viewer re-samples its local volume when the observation window changes; it does not merely enlarge a coarse full-domain grid.

![Native MATLAB full-field explorer](https://raw.githubusercontent.com/Liang-techh/OpenAI-NS-Velocity-Field-Reconstruction1/5d49ccf07d097a4bffeb35f6398936493b062d00/visualization/matlab/tests/output/matlab_explorer.png)

*Native MATLAB screenshot from the pinned upstream viewer test. It illustrates the same frozen ST054 field data. A screenshot is not a validation certificate. Streamlines are instantaneous curves, not material particle trajectories.*

See the [visualization guide](docs/VISUALIZATION.md) for controls, the selected viewing preset, central-core sampling, geometry diagnostics, and limitations.

**Verification boundary:** local Python tests, the installed wheel, and four full scientific replays have completed. The original full viewer has upstream native MATLAB tests. This release's modified core-view native tests are queued, not yet confirmed passed; the core view remains a new diagnostic tool pending that run.

## Current numerical results

Independent Cartesian finite-difference validation: 4,096 points per seed, six fixed times, separate spatial/time refinement and energy-quadrature ladders. Values below are worst over the six times at spatial step `0.005` and time step `0.0025`. Compare candidates **within the same seed**.

| Candidate | Seed | Full-vector sampled maximum | Spatial volume L2 | Release role |
|---|---|---:|---:|---|
| ST054-Q2 | 9175491 | 0.0373021893 | 0.0476190210 | Primary: lower L2 |
| ST054-Q2 | 9175492 | 0.0384553923 | 0.0477042271 | Primary: lower L2 |
| ST054-M3 | 9175491 | 0.0364076501 | 0.0483232046 | Lower-peak alternative |
| ST054-M3 | 9175492 | 0.0371248562 | 0.0478922083 | Lower-peak alternative |

Both momentum metrics must be below `0.001`. Neither candidate passes. The reported L2 is `sqrt(64 * mean(|R|^2))` at each time, **not** RMS or a space-time average. Sampled maxima are not continuous-domain upper bounds. Q2 is not better on every metric, and “primary” is not a global-best claim. ST055 is excluded because its complete frozen field/evidence bundle was not available for reproducible publication.

Sources: the original frozen-study summary restored to `evidence/upstream_ST054_results.json`, [release validation protocol](docs/VALIDATION.md), and the release replay reports produced by `tools/replay_release.py`.

## Use the Python API

Python 3.10+; install from the repository root:

```bash
python -m pip install -r requirements.txt
python tools/prepare_release.py  # One-time pinned setup; skips an intact initialized bundle
python -m pip install -e ".[test]"
ns-field verify
ns-field evaluate --candidate ST054-Q2 --point 0.1 0 0.1 --time 0.5
python -m pytest -q -W error
```

```python
from ns_reconstruction import load_candidate

field = load_candidate("ST054-Q2")
points = [[0.1, 0.0, 0.1], [0.0, 0.0, 0.2]]
u, p = field.fields(points, 0.5)
f = field.forcing(points, 0.5)
R = field.residual(points, 0.5)  # Analytic diagnostic, not independent validation
```

`fields` returns Cartesian velocity `(n, 3)` and pressure `(n,)`. Scalar or per-point times are accepted in `[0.25, 0.75]`. Evaluations are chunked to limit memory. The package includes its frozen data, so an installed wheel does not need a source checkout to evaluate a field.

For an independent full scientific check:

```bash
ns-field validate --candidate ST054-Q2 --seed 9175491 --out outputs/Q2_check.json
```

**Expected exit code: `1`**, reporting `momentum_max` and `momentum_L2` failures. Software and replay tests can pass while these scientific gates fail; the repository deliberately preserves that distinction.

## What is included

| Location | Contents |
|---|---|
| `src/ns_reconstruction/` | Installable API; pinned setup restores the frozen data and numerical runtime |
| `visualization/matlab/` | Corrected core tools; pinned setup restores the full-field viewer and native data |
| `evidence/` | Initialized bundles contain source/import records and four local replay reports; CI uploads new replays separately |
| `tests/` | Numerical, packaging, identity, input and English-documentation checks |
| `tools/` | Pinned import/replay tools; normal field/viewer use is offline |
| `docs/` | Model, validation, visualization, provenance and migration documentation |

## Physical contract and scientific boundaries

The field is axisymmetric with swirl, smooth and compactly supported in `r < 2`, `|z| < 2`. The physical domain is `R^3`, the evaluation box is `[-2, 2]^3`, viscosity is `0.01`, and the published window is `[0.25, 0.75]`. Initial kinetic energy is normalized to one. The prescribed force remains in the original bounded two-parameter divergence-free family; it is **not** defined by setting `f = R(u,p)` to manufacture a pass.

Pressure direction and flow-direction checks apply to explicitly declared finite core grids. They do not establish whole-domain source correspondence. Neither the original source's full oscillatory construction nor a matched heat exterior is claimed here. Moving a camera, changing a seed radius or zooming the core cannot improve the PDE residual.

Details: [model](docs/MODEL.md), [scientific status](docs/VALIDATION.md), [source and artifact provenance](docs/PROVENANCE.md).

## Repository setup and evidence

`tools/source_manifest.json` fixes each imported path, source commit and checksum. `prepare_matlab` initializes just the MATLAB components; `python tools/prepare_release.py` initializes the full Python/data/visualization release. Do not install an uninitialized source checkout and assume its external assets are present. The complete release ZIP contains the resolved files and local replay reports. CI independently repeats setup and validation, but queued jobs are not reported as completed.

## Repository transition

This is the replacement release for **`Liang-techh/OpenAI-NS-Velocity-Field-Reconstruction`** (without a trailing `1`). The previous contents remain in Git history and on `archive/pre-st054-release-20260919`. Historical branches and issues were not bulk-deleted or relabeled as completed. The separate research repository remains the experiment archive; this repository is the concise, runnable release.

See [migration notes](docs/MIGRATION.md). All current documentation, user-interface labels, comments and release metadata are in English.
