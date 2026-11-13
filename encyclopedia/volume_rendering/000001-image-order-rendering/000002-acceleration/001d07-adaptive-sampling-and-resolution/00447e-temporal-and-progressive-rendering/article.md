# Temporal and Progressive Rendering

Also called: Progressive Volume Rendering, Temporal Accumulation

## Overview
Temporal and Progressive Rendering reduces per-frame computational cost in volume rendering by distributing the sampling budget across multiple frames or by adaptively refining resolution based on prior results. It solves the problem of high rendering latency in real-time or interactive scenarios where full-resolution, high-sample-density rendering is infeasible. It distinguishes itself from siblings like [Two-stage NeRF Rendering](925-two-stage-nerf-rendering.md) and [Spatiotemporal Importance Sampling](4696-spatiotemporal-importance-sampling.md) by leveraging temporal coherence or iterative spatial refinement rather than static importance metrics or neural network inference [2872-temporal-upsampling-and-dithering.md][4836-checkerboard-progressive-rendering.md][7260-multi-resolution-rendering-for-clouds.md].

## Shared principles
The core mechanism shared by all children is the deviation from a fixed, uniform sampling strategy. Instead of casting a constant number of rays or samples per pixel/ray, the method computes a metric (such as gradient magnitude, rendering error, or temporal difference) and uses it to modulate the sampling rate or resolution.

Mathematically, this often involves a control loop:
1.  **Metric Evaluation:** Compute a scalar value $M$ representing local complexity or error.
2.  **Adaptation:** Adjust a parameter $P$ (step size $\Delta t$, resolution $R$, or sampling probability $w$) based on $M$.
3.  **Rendering/Training:** Execute the volume integral or loss computation with the adjusted parameter.

Common adaptation laws include:
*   **Step Size Adaptation:** $\Delta t \propto \frac{1}{M}$ or $\Delta t = \min(\Delta t_{max}, \max(\Delta t_{min}, \frac{k}{M}))$ [S6, S8, S9, unofficial].
*   **Resolution Control:** $R_{new} = R_{old} + \alpha \cdot (t_{target} - t_{actual})$ or threshold-based switching [S3063, unofficial].
*   **Probability Weighting:** $w_i \propto M_i$ for importance sampling [S4696, unofficial].

All methods in this family aim to maintain visual fidelity while reducing the total number of volume queries or network evaluations [S1][S2, unofficial].

## Subtopics

### [Temporal Upsampling and Dithering](2872-temporal-upsampling-and-dithering.md)
Combines spatial dithering with temporal accumulation to reduce noise in real-time volume rendering. It addresses the trade-off between spatial aliasing and temporal noise, differing from siblings by using full-resolution buffers with temporal blending rather than spatial subsampling or hierarchical structures.

### [Checkerboard Progressive Rendering](4836-checkerboard-progressive-rendering.md)
Reduces per-frame cost by tracing only half the pixels (checkerboard pattern) each frame and blending with previous results. It differs from siblings by using a fixed spatial subsampling pattern rather than temporal accumulation of full buffers or spatially varying resolution.

### [Multi-Resolution Rendering for Clouds](7260-multi-resolution-rendering-for-clouds.md)
Limits samples by rendering at low resolution first to identify areas needing refinement, then accumulating high-resolution results only in those regions. It differs from siblings by varying the spatial resolution of the rendering pass itself across iterations rather than relying on temporal coherence or fixed subsampling patterns.

## Background
The concept of progressive rendering in volume visualization dates back to early interactive ray marching, where reducing samples per frame allowed for real-time interaction [S1, unofficial]. Temporal accumulation techniques were adapted from real-time rasterization (e.g., TAA) to volume rendering to handle stochastic noise from sparse sampling [2872-temporal-upsampling-and-dithering.md]. Checkerboard patterns have been used in progressive image transmission and rendering to provide immediate visual feedback [4836-checkerboard-progressive-rendering.md]. Multi-resolution approaches for clouds emerged with the need to render large-scale volumetric effects efficiently, leveraging the fact that distant or low-opacity regions require less detail [7260-multi-resolution-rendering-for-clouds.md].

## Sources
[S1] Temporal and Progressive Rendering Notes, Unofficial, 2023
[S2] Adaptive Sampling Principles, Unofficial, 2023
[S6] Adaptive Step Size Control, Unofficial, 2023
[S8] Volume Rendering Error Metrics, Unofficial, 2023
[S9] Step Size Adaptation Laws, Unofficial, 2023
[S3063] Resolution Adaptation Control, Unofficial, 2023
[S4696] Importance Sampling Probability, Unofficial, 2023
