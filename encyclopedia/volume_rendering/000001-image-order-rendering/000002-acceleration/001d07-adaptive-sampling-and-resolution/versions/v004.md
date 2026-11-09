# Adaptive Sampling and Resolution

Also called: Adaptive Sampling, Adaptive Resolution, Adaptive Ray Marching

## Overview
Adaptive Sampling and Resolution is a family of volume rendering acceleration techniques that dynamically vary the computational effort—either by adjusting the number of samples per ray, the spatial resolution of the rendering grid, or the density of ray casting—based on local scene complexity, visual error, or performance budgets [S1][S2, unofficial]. It solves the problem of uniform computational waste in homogeneous regions or the excessive cost of high-fidelity rendering in complex areas by concentrating resources where they are most needed [S1][S2, unofficial]. It sets itself apart from static acceleration methods like [Early Z-Test Acceleration](8226-early-z-test-acceleration.md) or [Space Skipping and Empty Volume Culling](10063-space-skipping-and-empty-volume-culling.md) by actively modifying the sampling density or resolution parameters during the rendering or training process rather than merely skipping pre-defined empty regions [S1][S2, unofficial].

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

### [Two-stage NeRF Rendering](925-two-stage-nerf-rendering.md)
Decouples density estimation from color evaluation. A coarse network proposes samples in high-density regions, which are then importance-sampled by a fine network. It sets itself apart by explicitly separating the rough density estimation pass from the detailed evaluation pass. Choose when rendering neural radiance fields with significant empty space to avoid wasting fine-network queries on background.

### [Random Pixel Subsampling](1668-random-pixel-subsampling.md)
Selects a random subset of pixel coordinates for rendering or training. It solves high memory/compute costs by processing a sparse set of pixels. Unlike siblings, it relies on uniform random selection without spatial coherence, serving as a baseline for efficiency rather than quality. Choose for rapid prototyping or when spatial coherence is not required.

### [Temporal Upsampling and Dithering](2872-temporal-upsampling-and-dithering.md)
Applies blue noise dithering to low-resolution renders to break aliasing, then uses temporal accumulation to reconstruct high-resolution details. It distinguishes itself by leveraging temporal blending to mitigate noise, unlike static low-resolution rendering. Choose for interactive applications where frame-to-frame coherence can be exploited.

### [Resolution Adaptation](3063-resolution-adaptation.md)
Dynamically adjusts spatial discretization (voxel grid, viewport, or octree depth) based on performance or error metrics. It sets itself apart by focusing on the *structural* granularity of the representation rather than just per-ray sample counts. Choose when hardware constraints require dynamic resolution scaling or when rendering hierarchical data structures.

### [Adaptive Sampling Step with Early Ray Termination](4573-adaptive-sampling-step-with-early-ray-termination.md)
Decreases sampling frequency based on distance from the viewpoint and uses early ray termination. It requires a pre-integrated transfer function to maintain brightness consistency across varying step sizes. Choose for distant volumetric data where spatial coherence allows larger steps.

### [Spatiotemporal Importance Sampling](4696-spatiotemporal-importance-sampling.md)
Assigns higher sampling probabilities to pixels occluded by tools or exhibiting motion, and lower probabilities to stationary tissues. It solves inefficient training in surgical scenes by addressing varying learning difficulty across pixels. Choose for training neural fields on dynamic scenes with occlusions.

### [Checkerboard Progressive Rendering](4836-checkerboard-progressive-rendering.md)
Traces only half the pixels each frame using a deterministic checkerboard pattern, blending results over time. It sets itself apart from [Two-stage NeRF Rendering](925-two-stage-nerf-rendering.md) by using temporal blending of partial frames rather than spatial resolution stages. Choose for interactive applications requiring immediate feedback with progressive refinement.

### [Ray-Centric Adaptive Step and Termination](5938-ray-centric-adaptive-step-and-termination.md)
Dynamically adjusts sampling density along a ray based on local scene properties (e.g., gradient, opacity). It distinguishes itself by focusing on spatial adaptation along the ray trajectory rather than temporal frame budgeting. Choose for general volume rendering where local density varies significantly.

### [Multi-Resolution Rendering for Clouds](7260-multi-resolution-rendering-for-clouds.md)
Limits samples by rendering clouds at low resolution first to identify areas needing fewer samples, then accumulating with higher resolution passes. Choose for atmospheric rendering where large-scale structures dominate.

### [Adaptive Image Sampling with Pixel Sharing](8159-adaptive-image-sampling-with-pixel-sharing.md)
Casts rays only from the corners of square image regions and interpolates the interior, handling parallel synchronization by sharing corner pixels. It is suitable for MIMD shared-memory architectures. Choose for parallel volume rendering on shared-memory systems.

### [Context-Based Adaptive Ray Sampling with Quadtree Subdivision](8164-context-based-adaptive-ray-sampling-with-quadtree-subdivisio.md)
Partitions image space using a quadtree to allocate more rays to nontrivial areas (high error) and fewer to trivial areas. It operates continuously throughout training epochs, defining prior and posterior probability distributions to guide subdivision. Choose for training neural fields where error-driven refinement is beneficial.

### [Adaptive Sampling Rate Control](8896-adaptive-sampling-rate-control.md)
Not given in the sources.

### [Spatial Subdivision and Hierarchical Sampling](10997-spatial-subdivision-and-hierarchical-sampling.md)
Varies ray casting density across the image plane based on local visual complexity or error estimates using hierarchical structures (quadtrees). It solves uniform computational waste by concentrating effort on high-gradient regions. Choose for high-quality offline rendering where error bounds are critical.

#### Adaptive Feature Level Selection
- **Definition:** An anti-aliasing strategy that dampens high-resolution grid features based on the pixel footprint to prevent aliasing artifacts when rendering with adaptive sampling [S1].
- **Step:** Derive pixel radius $p(\mathbf{x})$ in contracted space for a given sample $\mathbf{x}$. Calculate optimal feature level $L(\mathbf{x})$ using the formula $L(\mathbf{x}) := -\log_2(2s \cdot p(\mathbf{x}))$. Multiply grid features at resolution level $L$ by the per-level weight $w_L$ calculated from the piecewise function [S1].
- **Formula:**
  $$ L(\mathbf{x}) := -\log_2(2s \cdot p(\mathbf{x})) $$
  where $s$ is the base grid resolution (128) [S1].
  $$ w_L = \begin{cases} 1 & \text{if } L < \lceil L(\mathbf{x}) \rceil \\ L(\mathbf{x}) - \lceil L(\mathbf{x}) \rceil & \text{if } \lceil L(\mathbf{x}) \rceil < L \leq \lceil L(\mathbf{x}) \rceil \\ 0 & \text{if } \lceil L(\mathbf{x}) \rceil < L \end{cases} $$
  These are per-level weights applied to grid features at resolution level $L$ [S1].
- **Parameter:** $s$: Base grid resolution, typically 128 [S1].

#### Adaptive Sparse Ray Sampling
- **Definition:** An optimization that casts a sparse grid of rays (less than one per pixel) and adaptively increases ray density in regions of high image complexity, exploiting coherence in projections [S2, unofficial].
- **Step:** Cast a sparse grid of rays, with density less than one ray per pixel. Measure local image complexity using color differences. Perform recursive subdivision of image space to increase sampling density in complex regions. Form images from the resulting nonuniform array of sample colors by interpolation and resampling at the display resolution [S2, unofficial].
- **Number:** Performance: Savings of more than two orders of magnitude over brute-force rendering methods were obtained for many datasets. Performance: Crude images obtained in a few seconds, with gradually better images at intervals of a few seconds each, culminating in high-quality images in less than a minute [S2, unofficial].

## Background
The concept of adaptive sampling in volume rendering originated from the need to handle the high computational cost of ray marching through dense volumetric data. Early work by [S2, unofficial] (DTIC ADA208091) described adaptive sparse ray sampling and recursive subdivision to exploit coherence in projections, achieving significant performance savings. More recently, with the rise of Neural Radiance Fields (NeRF), adaptive sampling has evolved to include importance sampling based on neural network outputs, such as the two-stage sampling in [925-two-stage-nerf-rendering.md] [S1]. The family also includes techniques for temporal coherence [2872-temporal-upsampling-and-dithering.md] and parallel execution [8159-adaptive-image-sampling-with-pixel-sharing.md].

## Sources
[S1] HybridNeRF: Efficient Neural Rendering via Adaptive Volumetric Surfaces, https://doi.org/10.1109/cvpr52733.2024.01858, fetched 2026-10-26
[S2] DTIC ADA208091: Interactive Visualization of 3D Medical Data, https://archive.org/details/DTIC_ADA208091, fetched 2026-10-26 (unofficial)
