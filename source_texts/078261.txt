# Gray-Scott 3D

An educational browser app for exploring three-dimensional Gray-Scott reaction-diffusion patterns.

日本語: 3次元 Gray-Scott 反応拡散モデルのパターンをブラウザで観察するための教育向け Web アプリです。

Live demo: https://gray-scott-3d.vercel.app/

## Overview

The Gray-Scott model is a reaction-diffusion system in which autocatalytic reaction terms and diffusion can generate rich spatial patterns. This app focuses on the three-dimensional case and renders regions where the concentration of `V` becomes visible.

The goal is public exploration rather than research-grade numerical analysis. The interface is designed for moving parameters, comparing presets, and seeing how 3D structures respond.

## Features

- 3D Gray-Scott simulation running in the browser
- Three.js / WebGL2 volume rendering
- Optional Marching Cubes isosurface view
- Switchable CPU Worker and WebGL2 GPGPU simulation backends with live timing metrics
- GPGPU is used by default when available, with automatic fallback to CPU mode
- Automatic render-quality profiles for phones, tablets, and desktop devices
- Manual `Auto`, `High quality`, and `Power saving` render modes
- Adjustable `F`, `k`, display threshold, simulation speed, and boundary condition
- Neumann and periodic boundary modes
- Reproducible initial conditions using a seeded Mersenne Twister
- Japanese / English UI with `AUTO`, `EN`, and `JP` language modes
- Separate guide page with equations, parameter notes, and references

## Model

The web app uses an explicit finite-difference Euler update for the Gray-Scott equations:

```text
U_t = Du laplacian(U) - U V^2 + F(1 - U)
V_t = Dv laplacian(V) + U V^2 - (F + k)V
```

`U` and `V` are concentrations, `Du` and `Dv` are diffusion coefficients, `F` is the feed rate, and `k` is the removal rate.

## Initial Conditions and Presets

Most presets start from a nearly uniform field with `U=1` and `V=0`, plus one or more small localized seeds. Seed radius is perturbed by deterministic random noise so the same seed value can reproduce the same initial state.

Some presets use multiple spherical or elongated seeds to make three-dimensional behavior easier to inspect.

## Visualization

The main view is GPU volume raymarching with Three.js and WebGL2.

1. The selected simulation backend updates the 3D `U` and `V` fields.
2. CPU mode runs the explicit stencil update in a Web Worker.
3. GPGPU mode packs `U,V` into compact tiled WebGL2 `RGBA32F` textures and advances them with a fragment shader and ping-pong framebuffers.
4. In GPGPU volume mode, the raymarching shader samples the current simulation texture directly, without per-frame GPU-to-CPU readback or `Data3DTexture` re-upload.
5. CPU mode uploads the quantized `V` field as a `THREE.Data3DTexture`.
6. A fragment shader accumulates color and opacity above the visible threshold.
7. Periodic CPU snapshots support the slice views, live metrics, and the Three.js `MarchingCubes` isosurface mode.

The UI reports sampled compute time, transfer/conversion time, and approximate cell-update throughput for comparing CPU and GPGPU behavior on the current browser and GPU. Marching Cubes remains CPU-bound, while the main GPGPU volume path stays on the GPU between periodic supporting-view snapshots.

Central XY, XZ, and YZ slice views are also shown as compact supporting views.

`Auto` quality starts phones with a lower pixel ratio and fewer raymarching samples, then observes actual frame timing and reduces quality further only when needed. GPGPU step batches are split into short chunks on mobile-class profiles so controls and camera movement can remain responsive without changing the numerical parameters or step sequence.

## Local Development

```bash
npm install
npm run dev
```

Vite usually serves the app at `http://127.0.0.1:5173/`. If that port is already in use, Vite may choose another port.

## Build

```bash
npm run build
npm run preview
```

The production build is written to `dist/`.

## Deployment on Vercel

This is a static Vite app. Import the repository into Vercel with the default Vite settings:

- Build Command: `npm run build`
- Output Directory: `dist`
- Install Command: `npm install`

The app has two static entry pages, `/` and `/about.html`; no custom rewrite configuration is required.

## Changelog

See [CHANGELOG.md](./CHANGELOG.md) for notable updates.

## Repository Structure

```text
.
├── index.html              # simulation UI
├── about.html              # model notes and references
├── public/                 # favicon, app icons, manifest
├── scripts/                # helper scripts
└── src/                    # application source
```

## References and Further Reading

- [1] P. Gray and S. K. Scott, ["Autocatalytic reactions in the isothermal, continuous stirred tank reactor"](https://doi.org/10.1016/0009-2509%2883%2980132-8), Chemical Engineering Science 38(1), 29-43 (1983).
- [2] P. Gray and S. K. Scott, ["Autocatalytic reactions in the isothermal, continuous stirred tank reactor: oscillations and instabilities in the system A+2B -> 3B, B -> C"](https://doi.org/10.1016/0009-2509%2884%2987017-7), Chemical Engineering Science 39(6), 1087-1097 (1984).
- [3] J. E. Pearson, ["Complex Patterns in a Simple System"](https://doi.org/10.1126/science.261.5118.189), Science 261(5118), 189-192 (1993).
- [4] Y. Nishiura and D. Ueyama, ["A skeleton structure of self-replicating dynamics"](https://doi.org/10.1016/S0167-2789%2899%2900010-X), Physica D: Nonlinear Phenomena 130(1-2), 73-104 (1999).
- [5] Y. Nishiura and D. Ueyama, ["Spatio-temporal chaos for the Gray-Scott model"](https://doi.org/10.1016/S0167-2789%2800%2900214-1), Physica D: Nonlinear Phenomena 150(3-4), 137-162 (2001).
- [6] T. Leppänen, M. Karttunen, K. Kaski, R. A. Barrio, and L. Zhang, ["A new dimension to Turing patterns"](https://doi.org/10.1016/S0167-2789%2802%2900493-1), Physica D: Nonlinear Phenomena 168-169, 35-44 (2002).
- [7] H. Shoji, K. Yamada, D. Ueyama, and T. Ohta, ["Turing patterns in three dimensions"](https://doi.org/10.1103/PhysRevE.75.046212), Physical Review E 75, 046212 (2007).
- [8] M. Munafo, ["Reaction-Diffusion by the Gray-Scott Model: Pearson's Parametrization"](https://www.mrob.com/pub/comp/xmorphia/index.html), a rich visual catalogue of two-dimensional Gray-Scott patterns. The 2D parameter map should not be treated as a direct map of 3D behavior.
- [9] Daishin Ueyama, ["The Gray-Scott モデル"](https://note.com/daishin_ueyama/n/na1798cca7752), note, January 30, 2025.
- [10] Daishin Ueyama, [Gray-Scott simulation results](https://youtube.com/playlist?list=PLmAO0Lzba3fx5wReOYxyHsIg9T8hmL7NB&si=wrdqE8UJD1cNDoDc), an older playlist of simulation results.

## License

MIT License.

Copyright (c) 2026 Daishin Ueyama.

See [LICENSE](./LICENSE) for details.
