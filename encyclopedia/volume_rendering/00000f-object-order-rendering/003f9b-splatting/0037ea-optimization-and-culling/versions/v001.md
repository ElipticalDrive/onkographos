# Optimization and Culling
## Overview
Optimization and Culling refers to techniques that reduce the computational workload of object-order volume splatting by skipping or discarding primitives that do not contribute significantly to the final image. It solves the performance bottleneck of processing large numbers of splats, particularly in real-time rendering. It sets itself apart from its siblings by focusing on pre-rasterization filtering (size culling) and screen-space occlusion checks (early termination) rather than geometric approximation or hardware-specific pipeline optimizations.
## Shared principles
The splatting family shares the fundamental mechanism of treating volumetric data as a set of geometric or parametric primitives that are transformed and rasterized [G].

1.  **Primitive Representation:** The volume is decomposed into primitives [G]. Each primitive $k$ is defined by its position $\mathbf{p}_k$, density/extinction coefficient $g_k$ or $\rho_k$, and a reconstruction kernel $h$ (typically a Gaussian) [S1][S6, unofficial][S10, unofficial].
2.  **Transformation:** Primitives are transformed from volume space to screen space using the Model-View-Projection (MVP) matrix or specific factorizations (e.g., shear-warp) [S2, S3, S4, unofficial]. For perspective projections, the kernel shape may be distorted (e.g., EWA splatting) to account for foreshortening [S6, unofficial][S10, unofficial].
3.  **Rasterization:** The transformed primitives are projected onto the 2D image plane [G]. The contribution of a voxel $D$ to a screen point $(x,y)$ is determined by the footprint function $F(x,y)$, which is the integral of the 3D kernel along the view direction [S2, unofficial][S6, unofficial][S10, unofficial]:
    $$ F(x,y) = \int h(x-x_D, y-y_D, z) \, dz $$
    The intensity written to the pixel is modulated by the voxel density:
    $$ P(x,y) = \text{weight}(x,y) \cdot \rho(D) $$
4.  **Compositing:** Contributions are accumulated to form the final image [G]. This is typically done using alpha blending in back-to-front or front-to-back order to handle transparency correctly [S1, S2, S3, unofficial][S7, unofficial]. Splatting compositing is performed along sheets (all pixels at once for a set of voxels), whereas ray-casting composites along rays (one sample at a time) [S10, unofficial].
5.  **Hardware Acceleration:** Most methods in this family rely on GPU rasterization pipelines for performance, enabling real-time interactivity [S1, S2, S3, unofficial].
## Subtopics

### [Early Splat Termination](811-early-splat-termination.md)
An optimization that skips the processing of voxels when the accumulated opacity at the target screen pixels is already full, analogous to early ray termination in ray casting. It sets itself apart from its siblings by operating in screen space to cull entire splats based on pixel coverage, whereas its nearest alternative, Early Ray Termination, operates in view space to terminate ray traversal. See [Early Splat Termination](811-early-splat-termination.md).

### [Splat Size Culling](2849-splat-size-culling.md)
A staging optimization that discards splats smaller than a defined threshold to reduce computational workload and improve real-time frame rates. It trades off visual quality for performance by ignoring splats whose contribution to the final image is deemed negligible, distinguishing itself from sibling methods like [Early Splat Termination](811-early-splat-termination.md) which operate during the rasterization phase rather than the data staging phase. See [Splat Size Culling](2849-splat-size-culling.md).

### Splat State Encoding
Storing a 32-bit integer for each splat where each potential state (highlighted, selected, deleted) is encoded inside a single bit, serving as an acceleration for transformation of selected splats [S1]. This allows the renderer to quickly check flags before loading an order of magnitude more of additional splat data.

## Background
Early Splat Termination was introduced as an optimization for object-order volume splatting to handle occlusion efficiently in screen space [S2, unofficial][S3]. Splat Size Culling is used in modern Gaussian Splatting pipelines to manage large datasets by filtering out negligible contributions during the staging phase [S1]. Splat State Encoding is part of the Splatshop framework for efficiently editing large Gaussian Splat models [S1].

## Sources
[S1] Splatshop: Efficiently Editing Large Gaussian Splat Models, https://doi.org/10.1111/cgf.70214, fetched 2026-10-31
