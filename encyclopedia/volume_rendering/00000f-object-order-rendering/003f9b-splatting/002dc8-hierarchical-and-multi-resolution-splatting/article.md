# Hierarchical and Multi-Resolution Splatting

Also called: Multi-Resolution Splatting, Hierarchical Splatting, LOD Splatting

## Overview
Hierarchical and Multi-Resolution Splatting optimizes the rendering of dense volumetric data and point clouds by organizing primitives into a spatial hierarchy, allowing the renderer to select primitives of varying granularity based on screen-space footprint or error metrics [S1, S2, unofficial][S3][S4, unofficial]. It solves the performance bottleneck of rendering high-density data by replacing clusters of fine primitives with single, aggregated coarse primitives in homogeneous or distant regions, while preserving detail in complex or foreground areas [S1, S2, unofficial][S3][S4, unofficial]. It distinguishes itself from sibling methods by its reliance on explicit error metrics or fixed downsampling rates to drive the selection of the appropriate resolution level, rather than relying solely on static distance thresholds or fixed grid subdivisions.

## Shared principles
The methods in this family share the mechanism of hierarchical decomposition and error-driven or distance-driven primitive selection, building on [Splatting](23-splatting.md).

1.  **Hierarchical Decomposition:** The input data (volume, point cloud, or Gaussian set) is organized into a tree structure (e.g., octree, pyramid, or sequential point tree) [S1, S2, unofficial][S3][S4, unofficial][S6, unofficial]. Each node in the hierarchy represents a spatial region and stores aggregate properties (mean position, average color/density, total opacity, or merged covariance) derived from its children [S1, S2, unofficial][S3][S4, unofficial].
2.  **Error Metric Computation:** For each internal node, a local error metric $E$ is computed to quantify the approximation cost of representing the node's children with the node's aggregate properties [S1, S2, unofficial][S3][S4, unofficial]. Common metrics include the Root Mean Square (RMS) error of scalar values or the deviation of child positions/opacities from the parent's mean [S1, unofficial][S3].
3.  **Adaptive Traversal and Selection:** During rendering, the hierarchy is traversed from the root to the leaves. A node is selected as a leaf (and thus rendered as a single primitive) if its error $E$ is below a user-defined threshold $\epsilon$ or if its projected screen-space footprint is smaller than a pixel threshold [S1, S2, unofficial][S3][S4, unofficial]. If the error exceeds the threshold, the algorithm recurses to the children [S1, S2, unofficial][S3][S4, unofficial].
4.  **Primitive Aggregation:** When a node is selected, its children are merged into a single representative primitive. The primitive's attributes (position, size, color, opacity) are computed as weighted averages or sums of the children's attributes [S1, S2, unofficial][S3][S4, unofficial]. For Gaussian splats, this involves merging covariances and spherical harmonics coefficients [S3].
5.  **Multi-Resolution Rendering:** The final image is composed of primitives from various levels of the hierarchy, allowing for efficient culling of distant or homogeneous regions while maintaining fidelity in foreground or complex areas [S1, S2, unofficial][S3][S4, unofficial][S6, unofficial].

## Subtopics

### [Multi Resolution Alpha Blending](12118-multi-resolution-alpha-blending.md)
Multi Resolution Alpha Blending is a fragment accumulation strategy used in hierarchical point splatting to resolve depth-ordering ambiguities when multiple splats project to the same pixel. It solves the problem of correct alpha compositing in scenes with varying point densities by collecting, sorting, and blending fragments on a per-pixel basis rather than globally. It differs from global sorting methods by operating on small, localized lists, which reduces computational overhead and memory bandwidth usage.

### [Multi-Resolution and Level of Detail (Pyramid, LOD, Downsampling)](12128-multi-resolution-and-level-of-detail-pyramid-lod-downsamplin.md)
Multi-Resolution and Level of Detail (LOD) techniques optimize volume and point cloud rendering by decomposing data into hierarchical structures, allowing the renderer to select primitives of varying granularity based on spatial error or screen-space footprint. This family solves the performance bottleneck of rendering high-density data by replacing clusters of fine primitives with single, aggregated coarse primitives in homogeneous or distant regions, while preserving detail in complex or foreground areas. It distinguishes itself from sibling methods by its reliance on explicit error metrics or fixed downsampling rates to drive the selection of the appropriate resolution level, rather than relying solely on static distance thresholds or fixed grid subdivisions.

### [Octree-based Hierarchy](12129-octree-based-hierarchy.md)
#### RBF Splatting with Octree and Fragment Programs
- **Definition:** A splatting implementation using Rational Basis Functions (RBFs) located by an octree, evaluated via fragment programs on cache-sheets [S1, unofficial].
- **Steps:**
    1. Add an octree to locate the Rational Basis Functions (RBFs) intersecting the cache-sheets [S1, unofficial].
    2. For each sheet plane pixel, use a fragment program to evaluate the exponential Gaussian function of the RBFs [S1, unofficial].
- **Performance:** 7 to 70 fps for Blunt-fin rendering at 400^2 resolution, caused by large variation of RBFs. Hardware: 2.8 GHz Pentium-IV, nVidia GeForce FX 5900 [S1, unofficial].

#### TreeSplat
- **Definition:** A method that organizes deformation as a mergeable tree, where each node represents a rigid part of an articulated object, and the overall deformation is computed by traversing the kinematic chain [S2, unofficial].

#### Depth-Aware Multi-Layer Splatting
- **Definition:** A splatting strategy that converts rasterized surface points into Gaussian splats and uses a multi-layer accumulation strategy based on depth to handle occlusion boundaries and disocclusions, producing smooth derivatives [S3].
- **Steps:**
    1. Render multiple layers of G-buffers. Maintain three accumulation buffers during splatting: $S^+$ for splats occluding the target pixel, $S^-$ for occluded splats, and $S^o$ for splats at the same depth as the target pixel [S3].
    2. To determine splat depth relation, pair up multi-layer surface intersections at splat center $p$ with closest intersections at target pixel $q$ in depth. The $p$ layer paired with the front-most $q$ layer is assigned to $S^o$, layers in front of it to $S^+$, and the rest to $S^-$ [S3].
    3. After all splats are rendered, buffers are separately normalized following Eq. 3 and composited in $S^-$, $S^o$, $S^+$ order using over-compositing to produce the final result $S$ [S3].
- **Pitfall:** Single-layer splatting treats all splats at the same depth, ignoring occlusions and producing spurious derivatives for occluded objects. Multi-layer splatting corrects this by distinguishing front, back, and coincident depths [S3].

#### Splat Downscaling Optimization
- **Definition:** Reduces Gaussian splat sizes to improve rendering speed while minimizing visual quality impact [S4, unofficial].
- **Parameter:** SplatDownscaleFactor: Float parameter in range [0, 1] controlling the degree of size reduction [S4, unofficial].

## Background
The concept of hierarchical splatting emerged alongside the need for real-time volume rendering. Early implementations focused on accelerating the evaluation of splats using spatial data structures. For instance, Jang et al. [Jang04] added an octree to locate rational basis functions (RBFs) intersecting cache-sheets, evaluating them via fragment programs [S1, unofficial]. This approach allowed for interactive frame rates on mid-sized datasets, though performance varied significantly based on the variation of RBFs [S1, unofficial]. Later, Neophytou et al. [Neophytou06] utilized floating-point rasterization facilities to avoid expensive fragment programs, achieving near-interactive frame rates on irregular data [S1, unofficial].

In the context of Gaussian Splatting, hierarchical methods have been adapted to handle dynamic deformations and multi-resolution rendering. TreeSplat, for example, organizes deformation as a mergeable tree for articulated objects [S2, unofficial]. Depth-aware multi-layer splatting was introduced to handle occlusion boundaries in differentiable rendering, ensuring accurate derivatives by distinguishing between occluding, occluded, and coincident splats [S3]. More recently, optimization techniques like Splat Downscaling have been integrated into rendering engines to reduce computational load by adjusting splat sizes [S4, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S3] Differentiable Surface Rendering via Non-Differentiable Sampling, https://doi.org/10.1109/iccv48922.2021.00603, fetched 2026-10-26
[S4] wuyize25/gsplat-unity, https://github.com/wuyize25/gsplat-unity, fetched 2026-10-27 (unofficial)
