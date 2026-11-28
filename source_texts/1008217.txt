# Noise Lab

Perlin, simplex, value and Worley noise with fBm octaves, ridged and turbulence modes and domain warping — seamlessly tileable, exportable as a PNG heightmap, a JSON grid or a GLSL function. Runs entirely in your browser.

**Live:** <https://noise-lab.slippylabs.com/>

## What it does

- Four generators: Perlin (gradient), simplex, value, and Worley in F1 and F2−F1 form.
- fBm, ridged and turbulence stacking with octaves, lacunarity, gain and domain warp.
- **Seamless tiling**, with the seam error measured and printed, and a 2×2 preview that makes a bad seam obvious.
- Terrain banding with draggable thresholds, and a hover readout of the value under the cursor.
- Export as a PNG heightmap, a 128×128 JSON grid, or a drop-in GLSL function.

## How it works

Tiling is done by making the **lattice** periodic, not by blending the edges. Each generator wraps its integer lattice coordinates through the period while keeping the true (unwrapped) offsets for the dot products — wrapping the offsets too would fold the surface and crease the seam. Because the unit square maps to exactly `frequency` lattice cells, the period *is* the frequency, and opposite edges are not merely similar: they are the same samples.

The fBm normaliser is the exact geometric sum of the octave amplitudes. Dividing by the octave count, which is the common shortcut, leaves the field washed out at low gain and clipped at high gain.

Simplex uses a triangular lattice that does not wrap onto a square. There is no seamless simplex, so the page refuses to tile it and says why, rather than producing a quiet seam.

## Verification

Coherent noise has no reference implementation worth diffing against — every library differs in gradient set and scaling. It has *defining properties*, and `verify_noise.py` tests those:

- **Gradient noise is exactly zero at every lattice point.** Measured over 289 integer points: **0.00e+00**. A control confirms value noise at the same points is not zero, so the test discriminates.
- With unit gradients, 2D Perlin is bounded by √2⁄2; 490k samples land inside [−0.874, 0.781] after the √2 normalisation.
- **Tiling: 192 combinations** of generator × fractal mode × frequency × octaves, every seam error **exactly 0.0**. The same field untiled has a seam error of 0.54.
- Worley's F1 and F2−F1 against a brute-force 7×7 feature-point search — far wider than the 3×3 the page searches, which is how that window is proved sufficient.
- Determinism per seed, the geometric normaliser, and a bounded gradient.

**266 checks.** Worley distances are measured in the query cell's own frame rather than in world coordinates; the absolute form was losing 2e-15 to rounding at large coordinates, which made a "tileable" field not quite bit-exact at its seam.
