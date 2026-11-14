# Level-of-Detail Selection
Also called: Image-based LOD selection

## Overview
Level-of-Detail (LOD) selection is a technique for rendering large volumetric datasets by dynamically choosing the appropriate resolution level to optimize the trade-off between image quality and computational cost [S1]. It solves the problem of rendering performance bottlenecks in large-scale volume visualization by ensuring that higher detail is only used where necessary [S1]. Unlike static multiresolution structures, this method employs an image-based quality metric to guide the selection process, distinguishing it from purely geometric or memory-based culling strategies [S1].

## How it works
The method relies on an image-based quality metric to determine the optimal LOD for each rendering pass or region [S1].

**Inputs:**
- Volumetric dataset $V$.
- Camera parameters (position, orientation, field of view).
- Target image resolution.
- Quality threshold or budget constraints.

**Outputs:**
- Selected LOD level for each ray or tile.
- Rendered image.

**Data Structures:**
- A hierarchy of volumetric representations (e.g., mipmaps, octrees) is assumed to be pre-computed or available, as implied by the need to select among levels [S1]. The specific layout of these levels is not detailed in the source, but they must support efficient access based on the quality metric [S1].

**Steps:**
1. **Ray Casting/Projection:** Cast rays through the volume or project the volume into the image space [S1].
2. **Quality Metric Calculation:** For each ray or image tile, calculate a quality metric that estimates the error or information loss if a lower LOD were used [S1]. The source does not specify the exact formula for this metric, only that it is image-based [S1].
3. **LOD Selection:** Compare the calculated quality metric against a threshold or budget. Select the highest LOD level that satisfies the quality constraint [S1].
4. **Rendering:** Render the volume using the selected LOD levels [S1].

**Parameters:**
- Quality threshold: Not given in the sources.
- Budget constraints: Not given in the sources.

**Edge Cases and Pitfalls:**
- The source does not specify handling for degenerate inputs or precision issues [S1].

## Subtopics

### Residual-Proxy Adaptive Selector
An adaptive LOD selection method that uses residual proxies to recover quality over fixed threshold methods at comparable primitive counts [S2, unofficial]. It is particularly effective in scenarios where sub-pixel redundancy is exhausted, such as in aerial models tessellated at their training viewing distance [S2, unofficial]. It is chosen when near-lossless quality is required with minimal primitive count overhead [S2, unofficial].

## Operating characteristics
The method improves image quality for large volumetric data by adapting the resolution to the visual importance [S1]. Performance characteristics are not explicitly quantified in the sources, but the use of an image-based metric suggests a computational overhead for quality estimation [S1]. The Residual-Proxy variant demonstrates significant quality recovery (0.49 dB) over fixed thresholds at comparable primitive counts (1.05x) on specific scenes [S2, unofficial].

## Background
The concept of using image-based quality metrics for LOD selection in volume rendering was proposed by Wang et al. [S1]. This approach was developed in the context of GPU-based raycasting pipelines to handle large volumetric data efficiently [S1]. The Residual-Proxy Adaptive Selector is described in the context of Budgeted-GS, a real-time large-scale Gaussian Splatting technique [S2, unofficial].

## Sources
[S1] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S2] Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD, https://arxiv.org/abs/2610.03162, fetched 2026-10-29 (unofficial)
