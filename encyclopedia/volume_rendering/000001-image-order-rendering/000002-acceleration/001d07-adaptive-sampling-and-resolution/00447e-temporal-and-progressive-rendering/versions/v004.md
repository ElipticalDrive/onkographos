# Temporal and Progressive Rendering


## Overview
Temporal and Progressive Rendering reduces per-frame computational cost in volume rendering by distributing the sampling budget across multiple frames or by adaptively refining resolution based on prior results [G]. It solves the problem of high rendering latency in real-time or interactive scenarios where full-resolution, high-sample-density rendering is infeasible [G]. It distinguishes itself from siblings like [Two-stage NeRF Rendering](925-two-stage-nerf-rendering.md) and [Spatiotemporal Importance Sampling](4696-spatiotemporal-importance-sampling.md) by leveraging temporal coherence or iterative spatial refinement rather than static importance metrics or neural network inference [2872-temporal-upsampling-and-dithering.md][4836-checkerboard-progressive-rendering.md][7260-multi-resolution-rendering-for-clouds.md].

## Shared principles
Builds on [Adaptive Sampling and Resolution](1412-adaptive-sampling-and-resolution.md).

## Subtopics

### [Temporal Upsampling and Dithering](2872-temporal-upsampling-and-dithering.md)
Combines spatial dithering with temporal accumulation to reduce noise in real-time volume rendering. It addresses the trade-off between spatial aliasing and temporal noise, differing from siblings by using full-resolution buffers with temporal blending rather than spatial subsampling or hierarchical structures. Choose it for real-time rendering with strict per-frame budgets where temporal coherence is available.

### [Checkerboard Progressive Rendering](4836-checkerboard-progressive-rendering.md)
Reduces per-frame cost by tracing only half the pixels (checkerboard pattern) each frame and blending with previous results. It differs from siblings by using a fixed spatial subsampling pattern rather than temporal accumulation of full buffers or spatially varying resolution. Choose it for interactive volume visualization requiring immediate partial feedback with minimal memory overhead.

### [Multi-Resolution Rendering for Clouds](7260-multi-resolution-rendering-for-clouds.md)
Limits samples by rendering at low resolution first to identify areas needing refinement, then accumulating high-resolution results only in those regions. It differs from siblings by varying the spatial resolution of the rendering pass itself across iterations rather than relying on temporal coherence or fixed subsampling patterns. Choose it for large-scale volumetric effects like clouds where detail is concentrated in specific regions.

### Progressive Refinement
A practical solution to rendering time by generating partial images that are progressively refined as the user interacts, using a pyramid data structure [S1, unofficial]. It supports both forward and backward viewing approaches [S1, unofficial]. Forward viewing renders the smaller volume in a pyramid data structure using large-footprint splats first, then higher resolution components [S1, unofficial]. Backward viewing first samples the screen in low resolution, interpolating values for regions where no rays were emitted, then casts more rays to replace interpolated values with accurate results [S1, unofficial]. Rays intended to cover large screen areas can be traced in lower-resolution pyramid components [S1, unofficial]. Sampling rate and stopping criteria can also be progressively refined [S1, unofficial].

### Render Pacer
A render pacer for each volume decides when a render shows the progress of a load, replacing the previous reRenderFraction/autoLoad mechanism to ensure a steady display rate independent of frame count [S2, unofficial]. The pacer requests a render immediately when data for a slice shown by the viewport is ready [S2, unofficial]. It requests a render when a retrieve stage completes (IMAGE_RETRIEVAL_STAGE) [S2, unofficial]. It requests a render when a timer expires during a load (e.g., after 150–250 ms) if new slices are ready [S2, unofficial]. It requests a render when the load completes [S2, unofficial]. A render started by the user does not wait for the pacer [S2, unofficial].

## Background
Progressive rendering in volume visualization addresses rendering time by generating partial images that are progressively refined as the user interacts [S1, unofficial]. Both forward and backward viewing approaches support this refinement [S1, unofficial]. In modern streaming contexts, a render pacer decides when a render shows the progress of a load, replacing previous mechanisms to ensure a steady display rate independent of frame count [S2, unofficial].

## Sources
[S1] Volume Visualization and Volume Rendering Techniques, https://www.cs.unh.edu/~cs880/visualization/Meissner-VolRenderingEGTutorial-rdb.pdf, fetched 2026-10-27 (unofficial)
[S2] Volume textures: brick copies in the render, a recorded update copy, paced renders, and a preload that states its loader, https://github.com/cornerstonejs/cornerstone3D/issues/2966, fetched 2026-10-27 (unofficial)
