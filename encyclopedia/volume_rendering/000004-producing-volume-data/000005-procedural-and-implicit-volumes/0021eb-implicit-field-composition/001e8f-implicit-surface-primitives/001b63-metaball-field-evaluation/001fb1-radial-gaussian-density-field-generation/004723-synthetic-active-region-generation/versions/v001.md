# Synthetic Active Region Generation
Also called: Mode C, Synthetic Bipole
## Overview
Generates a synthetic bipolar active region magnetogram using Gaussian flux concentrations, satellite flux, PIL shear, diffuse background, and noise [S1, unofficial]. It differs from [Particle Density Field Computation](4443-particle-density-field-computation.md) by explicitly modeling magnetic polarity (positive/negative bipoles) and shear effects rather than scalar particle density [S1, unofficial].
## How it works
**Inputs:** `seed` (int), `GRID_SIZE` (int, default 512 or similar).
**Outputs:** `bz` (2D array of shape `(GRID_SIZE, GRID_SIZE)`, dtype `float32`).

1.  **Initialize RNG and Grid:**
    *   `rng = np.random.default_rng(seed)`
    *   `g = GRID_SIZE`
    *   `x = np.linspace(-4.0, 4.0, g)`
    *   `y = np.linspace(-4.0, 4.0, g)`
    *   `XX, YY = np.meshgrid(x, y)`

2.  **Initialize Bz Grid:**
    *   `bz = np.zeros((g, g), dtype=np.float32)`

3.  **Add Primary Positive Bipole:**
    *   `pos_peak = rng.uniform(1200, 1800)`
    *   `pos_sigma = rng.uniform(0.60, 0.85)`
    *   `pos_cx, pos_cy = -1.35, 0.10`
    *   `bz += pos_peak * np.exp(-((XX - pos_cx)**2 + (YY - pos_cy)**2) / (2*pos_sigma**2))`

4.  **Add Primary Negative Bipole:**
    *   `neg_peak = -rng.uniform(900, 1400)`
    *   `neg_sigma = rng.uniform(0.75, 1.00)`
    *   `neg_cx, neg_cy = 1.50, -0.05`
    *   `bz += neg_peak * np.exp(-((XX - neg_cx)**2 + (YY - neg_cy)**2) / (2*neg_sigma**2))`

5.  **Add Satellite Flux:**
    *   `n_satellites = rng.integers(3, 7)`
    *   Loop `n_satellites` times:
        *   `sc_x, sc_y = rng.uniform(-3.0, 3.0), rng.uniform(-2.5, 2.5)`
        *   `sc_peak = rng.choice([-1, 1]) * rng.uniform(80, 350)`
        *   `sc_sig = rng.uniform(0.20, 0.50)`
        *   `bz += sc_peak * np.exp(-((XX - sc_x)**2 + (YY - sc_y)**2) / (2*sc_sig**2))`

6.  **Add PIL Shear:**
    *   `pil_shear = rng.uniform(0.08, 0.25)`
    *   `pil_warp = np.tanh((XX + pil_shear * YY) * 1.5) * rng.uniform(30, 80)`
    *   `bz += pil_warp * np.exp(-(XX**2 + YY**2) / 8.0)`

7.  **Add Diffuse Background:**
    *   `bz += rng.uniform(-40, 40) * np.exp(-(XX**2 + YY**2) / 12.0)`

8.  **Add Instrument Noise:**
    *   `bz += rng.normal(0, 15.0, (g, g)).astype(np.float32)`

9.  **Apply Boundary Apodisation:**
    *   `r2 = (XX / 4.2)**2 + (YY / 4.2)**2`
    *   `apod = np.clip(1.0 - r2, 0, 1)**0.5`
    *   `bz *= apod`

## Subtopics
## Operating characteristics
Not given in the sources.
## Background
Origin: [S1, unofficial].
## Sources
[S1] VitorFigueiredoResearch/AstroSplat-Browser-native-volumetric-visualization-for-astrophysical-vector-fields, https://github.com/VitorFigueiredoResearch/AstroSplat-Browser-native-volumetric-visualization-for-astrophysical-vector-fields, fetched 2026-10-26 (unofficial)
