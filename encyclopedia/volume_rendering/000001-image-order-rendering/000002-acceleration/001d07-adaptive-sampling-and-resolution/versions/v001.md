# Adaptive Sampling and Resolution


## Overview
Adaptive Sampling and Resolution is a family of volume rendering acceleration techniques that dynamically adjust computational effort based on local scene complexity or global performance budgets. It solves the problem of fixed-cost rendering by allocating more samples or higher resolution only where visually necessary, such as near boundaries or in high-variance regions, while skipping empty space or using lower fidelity elsewhere. It differs from its siblings by encompassing both per-ray sample count/step adjustments (e.g., [Adaptive Sampling](1385-adaptive-sampling.md), [Ray-Centric Adaptive Step and Termination](5938-ray-centric-adaptive-step-and-termination.md)) and global or pixel-level resolution changes (e.g., [Resolution Adaptation](3063-resolution-adaptation.md), [Random Pixel Subsampling](1668-random-pixel-subsampling.md)), whereas methods like [Early Termination and Probabilistic Path Control](1791-early-termination-and-probabilistic-path-control.md) focus primarily on stopping rays early without necessarily refining the remaining path or changing resolution.

## Shared principles
The family shares the core mechanism of varying the discretization of the rendering process—either in the number of samples per ray, the step size along the ray, the spatial resolution of the output, or the subset of pixels processed—to balance visual fidelity against computational cost.

1.  **Metric Evaluation**: A metric is computed to determine the required level of detail. This metric may be local (e.g., density $\sigma$, gradient magnitude $\nabla \sigma$, or color variance $\text{Var}(C)$ in a pixel neighborhood) or global (e.g., frame time, training iteration count) [S7, S5, S4].
2.  **Discretization Adjustment**: Based on the metric, the rendering parameters are adjusted:
    *   **Step Size**: The integration step $\Delta t$ is varied. For density-based methods, $\Delta t$ is often inversely proportional to density or gradient magnitude to maintain constant error bounds [S7, S5].
    *   **Sample Count**: The number of samples $K$ per ray is increased in high-detail regions and decreased in empty regions [S1].
    *   **Resolution**: The spatial resolution of the rendered image or the underlying voxel grid is adjusted [S1][S2, unofficial].
3.  **Integration/Rendering**: The volume rendering integral is computed using the adjusted discretization. For ray-centric methods, this involves a ray marching loop where $\Delta t$ changes at each step [S7, S5]. For resolution-based methods, this may involve rendering at a lower resolution and upsampling or subsampling pixels [S1][S2, unofficial].
4.  **Coherence and Stability**: To maintain GPU thread coherence, step sizes are often quantized or limited to a small set of discrete values, preventing divergence in warp execution [S5]. For temporal methods, dithering (e.g., blue noise) is used to break up coherent aliasing patterns, and temporal accumulation is used to smooth noise [S1].

## Subtopics

### [Two-stage NeRF Rendering](925-two-stage-nerf-rendering.md)
Two-stage NeRF Rendering evaluates a scene using two distinct neural network passes per ray. It differs from its siblings by explicitly separating the sampling strategy into a coarse pass and a fine pass with fixed sample counts, whereas [Adaptive Sampling](1385-adaptive-sampling.md) adjusts sample counts based on variance, and [Random Pixel Subsampling](1668-random-pixel-subsampling.md) skips pixels entirely rather than refining them. It is chosen when high-frequency detail is required but a single pass with many samples is too expensive or when the neural network benefits from a coarse-to-fine refinement strategy.

### [Random Pixel Subsampling](1668-random-pixel-subsampling.md)
Random Pixel Subsampling selects a random subset of pixel coordinates from a full image grid to reduce the computational cost of training volume rendering models, such as Neural Radiance Fields (NeRFs), by evaluating the scene only at a fraction of the pixels per iteration. It differs from [Two-stage NeRF Rendering](925-two-stage-nerf-rendering.md) by operating within a single training pass using stochastic gradient descent on a pixel subset, rather than separating coarse and fine rendering stages, and from [Resolution Adaptation](3063-resolution-adaptation.md) by keeping the image resolution constant while varying the number of active pixels. It is chosen to accelerate training by reducing the number of forward passes per iteration.

### [Temporal Upsampling and Dithering](2872-temporal-upsampling-and-dithering.md)
Temporal Upsampling and Dithering is a rendering technique that combines spatial resolution reduction with temporal accumulation to mitigate sampling artifacts in volume rendering, specifically addressing slicing artifacts caused by insufficient sampling steps. It distinguishes itself from standard adaptive sampling methods by using blue noise dithering to break up coherent aliasing patterns and temporal upsampling to reduce the noise introduced by that dithering, rather than relying solely on spatial supersampling or static resolution scaling. It is chosen for real-time rendering where performance budgets are tight but temporal coherence can be exploited.

### [Resolution Adaptation](3063-resolution-adaptation.md)
Resolution Adaptation dynamically adjusts the spatial resolution of the volumetric representation or the rendering viewport during execution to balance visual fidelity against computational cost. It solves the problem of fixed-resolution rendering wasting resources on empty space or failing to meet performance budgets in complex regions. It differs from [Adaptive Sampling](1385-adaptive-sampling.md) by modifying the grid or viewport granularity rather than the number of samples per ray, and from [Level-of-detail (LOD) methods](1007-level-of-detail-lod-methods.md) by often modifying the underlying data structure or training state rather than just swapping pre-computed assets. It is chosen when the scene complexity varies significantly across the view or when strict frame time budgets must be met.

### [Ray-Centric Adaptive Step and Termination](5938-ray-centric-adaptive-step-and-termination.md)
Ray-Centric Adaptive Step and Termination adjusts the distance between samples along a ray based on local volume properties (such as density or gradient magnitude) to skip empty space and refine detail near boundaries. It differs from [Adaptive Sampling](1385-adaptive-sampling.md) by dynamically changing the step size $\Delta t$ within a single ray cast rather than changing the number of rays or blocks, and from [Two-stage NeRF Rendering](925-two-stage-nerf-rendering.md) by operating on explicit volumetric data structures rather than neural uncertainty estimates. It is chosen for explicit volume rendering where density gradients are available and efficient skipping of empty space is critical.

### Mixed Resolution Scene Rendering
- **Definition**: Combines full-resolution scene elements with low-resolution high-density volume data by rendering the volume to off-screen targets and projecting them onto a quad [S1].
- **Steps**:
    1.  Setup two off-screen cameras, one for each eye, updating each frame [S1].
    2.  Setup two low-resolution render targets (e.g., 200x200) for the cameras [S1].
    3.  Set up a quad that moves in front of the user [S1].
    4.  Each frame: Draw the low-resolution render targets for each eye (containing volume data and expensive shaders) [S1].
    5.  Each frame: Draw the scene normally at full resolution (meshes, UI) [S1].
    6.  Each frame: Draw a quad in front of the user and project the low-res renders onto it [S1].
- **When to choose**: When rendering mixed scenes with both high-detail meshes and high-density volumes, allowing the volume to be rendered at a lower cost while maintaining full resolution for other elements [S1].

### Checkerboard Progressive Rendering
- **Definition**: Reduces per-frame compute by tracing only half the pixels each frame while blending results over time [S2, unofficial].
- **Steps**:
    1.  Alternate which pixels are freshly traced each frame [S2, unofficial].
    2.  Non-traced pixels carry forward the previous frame's blended result [S2, unofficial].
- **When to choose**: When real-time performance is critical and temporal coherence can be leveraged to hide the reduced spatial sampling [S2, unofficial].

### Adaptive Feature Level Selection
- **Definition**: An anti-aliasing strategy that dampens high-resolution grid features based on the pixel footprint to prevent aliasing artifacts when rendering with adaptive sampling [S3].
- **Steps**:
    1.  Derive pixel radius $p(\mathbf{x})$ in contracted space for a given sample $\mathbf{x}$ [S3].
    2.  Calculate optimal feature level $L(\mathbf{x})$ using the formula $L(\mathbf{x}) := -\log_2(2s \cdot p(\mathbf{x}))$ [S3].
    3.  Multiply grid features at resolution level $L$ by the per-level weight $w_L$ calculated from the piecewise function [S3].
- **Formula**:
    $$L(\mathbf{x}) := -\log_2(2s \cdot p(\mathbf{x}))$$
    where $L(\mathbf{x})$ is the optimal feature level for sample $\mathbf{x}$, $s$ is base grid resolution (128), and $p(\mathbf{x})$ is the pixel radius in contracted space [S3].
    $$w_L = \begin{cases} 1 & \text{if } L < \lceil L(\mathbf{x}) \rceil \\ L(\mathbf{x}) - \lceil L(\mathbf{x}) \rceil & \text{if } \lceil L(\mathbf{x}) \rceil < L \leq \lceil L(\mathbf{x}) \rceil \\ 0 & \text{if } \lceil L(\mathbf{x}) \rceil < L \end{cases}$$
    These are per-level weights applied to grid features at resolution level $L$ [S3].
- **Parameters**: $s$: Base grid resolution, typically 128 [S3].
- **When to choose**: When rendering hierarchical grid-based volumes (e.g., NeRFs with multi-resolution hash grids) and anti-aliasing is required to prevent high-frequency artifacts [S3].

### Adaptive Screen Sampling
- **Definition**: Introduced by Levoy [Levoy-1990-HRT], this method adaptively switches sampling rates based on color change in a pixel neighborhood, tracing additional rays in areas with high value gradients [S4, unofficial].
- **Steps**:
    1.  Trace additional rays in areas with high value gradients. If rays miss the surface, interpolate missing values [S4, unofficial].
    2.  Switch to higher sampling rate to replace interpolated values with exact color values from a second sampling step [S4, unofficial].
- **When to choose**: In traditional ray tracing or volume rendering where adaptive sampling is needed to handle high-frequency color changes efficiently [S4, unofficial].

## Background
Adaptive sampling and resolution techniques have evolved from early ray tracing methods to modern neural rendering pipelines. Adaptive screen sampling was introduced by Levoy in 1990 for traditional ray tracing, adapting sampling rates based on color gradients [S4, unofficial]. Mixed resolution rendering has been used in mixed reality applications to balance the cost of high-density volume rendering with full-resolution scene elements [S1]. In the context of Neural Radiance Fields (NeRFs), two-stage rendering was proposed to improve quality by refining coarse samples with a fine network [S1]. Random pixel subsampling has been adopted to accelerate NeRF training by reducing the number of pixels processed per iteration [S1][S2, unofficial]. Temporal upsampling and dithering techniques, such as those using blue noise, have been adapted for volume rendering to mitigate slicing artifacts and improve performance [S1]. Adaptive feature level selection has been developed for grid-based NeRFs to handle anti-aliasing in multi-resolution representations [S3].

## Sources
[S1] Volume rendering - Mixed Reality | Microsoft Learn, https://learn.microsoft.com/en-us/windows/mixed-reality/develop/native/volume-rendering, fetched 2026-10-27
[S2] NiekMSoftware/bad-apple-voxel-raytracer, https://github.com/NiekMSoftware/bad-apple-voxel-raytracer, fetched 2026-10-24 (unofficial)
[S3] HybridNeRF: Efficient Neural Rendering via Adaptive Volumetric Surfaces, https://doi.org/10.1109/cvpr52733.2024.01858, fetched 2026-10-26
[S4] [PDF] Direct Volume Rendering | CGL @ ETHZ, https://cgl.ethz.ch/teaching/former/scivis_07/Notes/stuff/StuttgartCourse/VIS-Modules-06-Direct_Volume_Rendering.pdf, fetched 2026-10-26 (unofficial)
