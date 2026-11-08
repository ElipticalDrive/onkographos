# Ray Traversal Schemes

Also called: Accumulate Projection, Depth Projection, Max Intensity Projection, Ray Traversal -Accumulate

## Overview
Ray traversal schemes define how scalar voxel values along a ray are aggregated into a single pixel color and opacity, ranging from simple statistical projections to full volumetric compositing [S1, S2, S3, unofficial]. Unlike standard front-to-back ray casting which integrates light transport, these schemes often prioritize specific visual features (such as maximum intensity or surface proximity) or differentiability for inverse graphics [S2, S4, unofficial]. They are distinguished from [Ray Marching and Discretization](2788-ray-marching-and-discretization.md) by their focus on the aggregation logic rather than the sampling interval, and from [Monte Carlo Path Tracing for Volume Rendering](4980-monte-carlo-path-tracing-for-volume-rendering.md) by their deterministic, single-pass nature.

## Shared principles
All methods in this family operate on the ray segment $[t_{start}, t_{end}]$ defined by the intersection with the volume domain. For each sample point $\mathbf{p}_i$ along the ray, a scalar value $d_i$ is obtained via interpolation (typically trilinear) from the voxel grid [S1, S2, S3, S4, unofficial]. This scalar is mapped to optical properties (color $C_i$ and opacity $\alpha_i$) via a transfer function [S2, S3, unofficial]. The core distinction lies in the aggregation function $A(\{(C_i, \alpha_i)\}_{i=1}^N)$ that combines these samples into the final pixel color $C_{final}$ and opacity $\alpha_{final}$.

## Subtopics

### Intensity Projection
Projects the raw scalar intensity value of the voxel with the highest magnitude along the ray. It does not use opacity or compositing.
*   **Mechanism:** $C_{final} = \max_i (d_i)$.
*   **Use case:** Highlighting high-density structures.
*   **Link:** [Intensity Projection](52-intensity-projection.md)

### Max Intensity Projection (MIP)
Projects the maximum intensity value along the ray, effectively ignoring occlusion. It is widely used in medical imaging to visualize high-contrast structures like blood vessels.
*   **Mechanism:** $C_{final} = \max_i (d_i)$, often mapped to a color table.
*   **Use case:** Magnetic Resonance Angiogram (MRA) and CT angiography [S2, S3, unofficial].
*   **Link:** [Max Intensity Projection](53-max-intensity-projection.md)

### Average Projection
Produces an X-ray-like picture by averaging the scalar values along the ray. It provides a global view of density but lacks depth cues.
*   **Mechanism:** $C_{final} = \frac{1}{N} \sum_{i=1}^N d_i$.
*   **Use case:** Quick overview of volume density.
*   **Link:** [Average Projection](54-average-projection.md)

### Accumulate Projection
Accumulates opacity while compositing colors to make transparent layers visible. This is the standard front-to-back volume rendering approach.
*   **Mechanism:** Uses the front-to-back compositing equations:
    $$ C'_{i} = C'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i C_i $$
    $$ \alpha'_{i} = \alpha'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i $$
    where $C'_0 = 0$ and $\alpha'_0 = 0$.
*   **Use case:** General volume visualization where transparency and internal structure are important.
*   **Link:** [Accumulate Projection](55-accumulate-projection.md)

### First Projection
Extracts the iso-surface closest to the camera. It identifies the first voxel that exceeds a certain threshold or opacity.
*   **Mechanism:** Finds the first index $i$ where $\alpha_i > \tau$ (or $d_i > \tau$) and returns its color/depth.
*   **Use case:** Surface extraction and depth estimation.
*   **Link:** [First Projection](56-first-projection.md)

### Depth Projection
Projects the distance $t$ along the ray to the point of interest (e.g., the first surface hit or the center of mass).
*   **Mechanism:** Returns $t_{final}$ corresponding to the selected sample.
*   **Use case:** Depth maps and spatial reasoning.
*   **Link:** [Depth Projection](57-depth-projection.md)

### Voxel Aggregation Methods
A set of differentiable aggregation techniques used in inverse graphics and neural rendering, where the goal is to render images that are differentiable with respect to the input voxel data.

#### Occupancy Probability Aggregation
Yan et al. associate an occupancy probability to each pixel. This value is computed by casting a ray from the pixel, sampling all associated voxels, and choosing the one with the highest occupancy probability [S4, unofficial].

#### Stopping Probability Aggregation
Tulsiani et al. compute the probability that a ray stops at a certain distance. This allows rendering of depth maps, color images, and foreground masks of an object [S4, unofficial].

#### Bidirectional Emission-Absorption
Henzler et al. introduce a bidirectional emission-absorption (EA) light radiation model for emitting materials, in addition to visual hull (VH) and absorption-only (AO) models similar to Tulsiani et al. [S4, unofficial].

#### Iterative Accumulation with Discarding
Lombardi et al. employ differentiable ray marching by casting a ray from each pixel and iteratively accumulating color and opacity probabilities along it. When the cumulative sum of the opacity values along the ray reaches a maximum value, the remaining voxels are discarded to prevent further impact on the color [S4, unofficial].

#### Surface-Weighted Aggregation
Jiang et al. handle voxels that represent signed distance functions. They cast a ray from the pixel and locate the surface voxel closest to the camera. The final pixel color is computed as a weighted sum of the surface voxel and its neighboring voxels. A normal vector is computed at the surface point based on voxel values around it for shading. Although locating the surface point is not a differentiable operation, the pixel color is differentiable with respect to the voxels around the surface point [S4, unofficial].

## Background
Ray traversal schemes originated with the development of Direct Volume Rendering (DVR). The "Accumulate" scheme, based on front-to-back compositing, was formalized by Levoy in 1988 [S1, S2, S3, unofficial]. MIP and Average projections were developed as faster, feature-specific alternatives for medical imaging, particularly for visualizing high-contrast structures like blood vessels in Magnetic Resonance Angiography [S2, S3, unofficial]. Recent developments in differentiable rendering have expanded these schemes to include probabilistic and surface-weighted aggregation methods to support gradient-based optimization [S4, unofficial].

## Sources
[S1] PPT - Direct Volume Rendering (DVR): Ray-casting PowerPoint Presentation - ID:465503, https://slideserve.com/jariath/direct-volume-rendering-dvr-ray-casting, fetched 2026-10-24 (unofficial)
[S2] Direct Volume Rendering Techniques, https://www.scribd.com/presentation/380929057/10-raycasting, fetched 2026-10-25 (unofficial)
[S3] Direct Volume Rendering (DVR): Ray-casting | PPT - SlideShare, https://www.slideshare.net/slideshow/direct-volume-rendering-dvr-raycasting/265104120, fetched 2026-10-26 (unofficial)
[S4] Differentiable Rendering: A Survey, https://doi.org/10.48550/arxiv.2006.12057, fetched 2026-10-26 (unofficial)
