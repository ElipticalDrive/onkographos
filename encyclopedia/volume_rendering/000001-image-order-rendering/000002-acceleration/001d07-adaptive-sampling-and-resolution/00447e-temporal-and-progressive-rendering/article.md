# Temporal and Progressive Rendering

Also called: Temporal Accumulation, Progressive Refinement, Checkerboard Rendering

## Overview
Temporal and Progressive Rendering accelerates volume rendering by trading spatial completeness for temporal coherence, computing only a subset of pixels or samples per frame and accumulating results over time. It solves the problem of high computational cost in interactive or real-time scenarios by allowing the image to converge gradually rather than waiting for a full, noise-free render. It distinguishes itself from static adaptive methods by leveraging frame-to-frame history to reduce noise and from pure dithering by reconstructing high-frequency details through temporal integration rather than spatial oversampling.

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
A technique that renders at reduced resolution with blue noise dithering to break aliasing, then uses temporal upsampling to reconstruct high-resolution details. It differs from static low-resolution rendering by leveraging temporal accumulation and from pure dithering by mitigating noise through frame blending.

### [Checkerboard Progressive Rendering](4836-checkerboard-progressive-rendering.md)
Reduces per-frame compute by tracing only half the pixels (checkerboard pattern) each frame and blending results over time. It sets itself apart from [Two-stage NeRF Rendering](925-two-stage-nerf-rendering.md) by using temporal blending of partial frames rather than spatial resolution stages, and from [Random Pixel Subsampling](1668-random-pixel-subsampling.md) by using a deterministic spatial pattern.

### [Multi-Resolution Rendering for Clouds](7260-multi-resolution-rendering-for-clouds.md)
Limits total volume samples by initially rendering at low resolution to identify regions requiring detail, then accumulating the final image with higher-resolution passes. Unlike standard adaptive sampling which varies sample counts per pixel, this method varies the spatial resolution of the entire rendering pass, using the coarse pass to skip expensive evaluations in empty areas.

## Background
Temporal accumulation and progressive refinement are foundational concepts in real-time graphics, originally developed for rasterization and ray tracing to achieve interactive frame rates. In the context of volume rendering and neural radiance fields (NeRF), these techniques have been adapted to handle the high cost of volumetric integration and neural network inference. The specific application of checkerboard patterns and temporal upsampling for volume rendering has emerged as a solution to the "frozen" or noisy appearance of low-sample renders during camera interaction.

## Sources
[S1] Temporal and Progressive Rendering Notes, Unofficial
[S2] Neural Radiance Fields Book, Chapter on Rendering
[S6] Adaptive Step Size Control in Volume Rendering
[S8] Error Metric Based Sampling
[S9] Step Size Adaptation Laws
[S3063] Resolution Adaptation Notes
[S4696] Spatiotemporal Importance Sampling Notes
