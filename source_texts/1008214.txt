# WebGL Animated Gradient

A self-contained, dependency-free animated gradient background in raw WebGL/GLSL.
Domain-warped fBm produces an organic, painterly color flow that reacts subtly to
the cursor and stays crisp at any viewport size, including 4K.

**Live demo → https://axgord.github.io/webgl-gradient/**

## Highlights

- **No dependencies, no build** — single `index.html`, drop-in anywhere (incl. WordPress).
- **Procedural** — domain-warped fractal noise (Inigo Quilez technique), not baked frames.
- **Tuned for the GPU:**
  - `sin()`-free polynomial hash (also fixes precision on mobile `highp`).
  - Tiered octaves — warp layers stay low-frequency; full detail only in the final lookup.
  - `vec2`-valued noise: both warp channels share one lattice and corner hashes.
- **Resolution-independent** with DPR cap, responsive resize, and a CSS fallback when WebGL is absent.
- Built-in FPS / frame-time / 1%-low overlay (remove the demo overlays for production).

## Use

Open `index.html`, or copy the `<canvas>` + the shader/JS block into your page.
Tune palette and motion via the constants in the fragment shader.
