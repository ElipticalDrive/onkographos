# Hierarchical and Multi-Resolution Splatting

Also called: Multi-Resolution Splatting, LOD Splatting

## Overview
Hierarchical and Multi-Resolution Splatting addresses the performance bottleneck of rendering large volumetric datasets or dense point clouds by adapting the rendering resolution to the viewer's distance and desired quality. It sets itself apart from standard splatting by using a multi-resolution structure (pyramid, octree, or list) to select appropriate detail levels, and from [Splatting Without the Blur](1157-splatting-without-the-blur.md) by explicitly managing spatial hierarchy rather than just kernel shape.

## Shared principles
The family shares the mechanism of decomposing the input data into a hierarchy of representations, where coarser levels approximate finer levels. The core principle is **Level-of-Detail (LOD) Selection**: during rendering, the system determines which level of the hierarchy to use for a given spatial region or pixel based on a metric (e.g., screen-space footprint, error threshold, or occlusion).

1.  **Hierarchical Construction:** The input (voxels, points, or Gaussians) is organized into a tree or pyramid.
    *   **Voxel Pyramids:** Created by averaging $2 \times 2 \times 2$ blocks of the previous level [S1, S2, unofficial].
    *   **Point/Octree Aggregation:** Internal nodes store aggregate properties (mean position, color, size) of their children [S1, unofficial].
    *   **Gaussian Merging:** Internal nodes store a "merged" Gaussian with weighted mean position, covariance, and opacity [S3, unofficial].
2.  **Selection Metric:** A criterion determines if a node is sufficiently accurate or small enough to be rendered as a single primitive.
    *   **Error Threshold:** Root mean square (RMS) error between the node's aggregate and its children [S1, unofficial].
    *   **Screen-Space Footprint:** The projected size of the primitive relative to pixel size [S3, unofficial].
    *   **Occlusion:** Visibility maps or depth buffers cull nodes that are fully occluded [S1, unofficial][S4].
3.  **Rendering:** The selected primitives are splatted using the parent family's rasterization and compositing rules [Parent](23-splatting.md).

## Subtopics

### [Hierarchical Splatting](1168-hierarchical-splatting.md)
A progressive refinement algorithm using a pyramidal volume representation. It constructs a pyramid of averaged volume cells and fits an octree based on a precision threshold. It is chosen when rendering large static volumetric datasets where interactive frame rates are required. It differs from [Multi-Resolution Points](4784-multi-resolution-points.md) by using averaged voxel cells rather than point aggregation.

### [Multi-Resolution Points](4784-multi-resolution-points.md)
Organizes points into an octree and uses a local RMS error metric to decide on point size, allowing the substitution of fine-grained primitives with coarser aggregates. It is chosen for dense point cloud rendering. It differs from [Hierarchical Splatting](1168-hierarchical-splatting.md) by operating on explicit points and using an octree with error-based pruning rather than a pre-computed voxel pyramid.

### [3D Gaussian Splatting LOD methods](5439-3d-gaussian-splatting-lod-methods.md)
Addresses the high memory footprint of monolithic 3D Gaussian representations by reducing redundancy, compressing attributes, or organizing primitives into hierarchical structures. It is chosen for large-scale 3DGS scenes. It differs from [Hierarchical Splatting](1168-hierarchical-splatting.md) by targeting the unique attribute structure of 3D Gaussians (position, scale, rotation, opacity, SH) rather than generic voxels.

### [Splat List (SVL) Generation with Multi-Level Downsampling](5767-splat-list-svl-generation-with-multi-level-downsampling.md)
Generates a Sparse Voxel List (SVL) of splat geometry across multiple LODs by iteratively downsampling a voxel occupancy mask. It is chosen for efficient GPU upload of voxel-based splats. It differs from [Hierarchical Splatting](1168-hierarchical-splatting.md) by explicitly constructing a flat, contiguous buffer of splat indices organized by LOD, rather than relying on tree traversal during rasterization.

### RBF Splatting with Octree and Fragment Programs
A splatting implementation using Rational Basis Functions (RBFs) located by an octree, evaluated via fragment programs on cache-sheets [S1, unofficial].
*   **Step:** Add an octree to locate the RBFs intersecting the cache-sheets. For each sheet plane pixel, use a fragment program to evaluate the exponential Gaussian function of the RBFs [S1, unofficial].
*   **Performance:** 7 to 70 fps for Blunt-fin rendering at $400^2$ resolution, caused by large variation of RBFs. Hardware: 2.8 GHz Pentium-IV, nVidia GeForce FX 5900 [S1, unofficial].

### Hierarchical Splat Culling
Splats are culled prior to sorting and rendering using a custom octree to optimize performance [S2, unofficial].

### TreeSplat
A method that organizes deformation as a mergeable tree, where each node represents a rigid part of an articulated object, and the overall deformation is computed by traversing the kinematic chain [S3, unofficial].

### Depth-Aware Multi-Layer Splatting
A splatting strategy that converts rasterized surface points into Gaussian splats and uses a multi-layer accumulation strategy based on depth to handle occlusion boundaries and disocclusions, producing smooth derivatives [S4].
*   **Step:** Render multiple layers of G-buffers. Maintain three accumulation buffers during splatting: $S^+$ for splats occluding the target pixel, $S^-$ for occluded splats, and $S^o$ for splats at the same depth as the target pixel [S4].
*   **Step:** To determine splat depth relation, pair up multi-layer surface intersections at splat center $p$ with closest intersections at target pixel $q$ in depth. The $p$ layer paired with the front-most $q$ layer is assigned to $S^o$, layers in front of it to $S^+$, and the rest to $S^-$ [S4].
*   **Step:** After all splats are rendered, buffers are separately normalized following Eq. 3 and composited in $S^-$, $S^o$, $S^+$ order using over-compositing to produce the final result $S$ [S4].
*   **Pitfall:** Single-layer splatting treats all splats at the same depth, ignoring occlusions and producing spurious derivatives for occluded objects. Multi-layer splatting corrects this by distinguishing front, back, and coincident depths [S4].

### Splat Downscaling Optimization
Reduces Gaussian splat sizes to improve rendering speed while minimizing visual quality impact [S5, unofficial].
*   **Parameter:** SplatDownscaleFactor: Float parameter in range [0, 1] controlling the degree of size reduction [S5, unofficial].

### Octree-based Splatting
*   **Step:** Change the size of a splat based upon the cell it represents in an octree representation of the volume [S6, unofficial].
*   **Step:** Cull voxels using an octree structure to accelerate rendering [S6, unofficial].

## Background
The concept of hierarchical rendering for volumes dates back to the mid-1990s. Cabral et al. [Cabral94] were among the first to demonstrate the use of texture mapping hardware for splatting. Laur and Hanrahan [8] introduced the use of octrees to change splat size and cull voxels, accelerating rendering without reducing quality [S6, unofficial]. Jang et al. [Jang04] added an octree to locate Rational Basis Functions (RBFs) intersecting cache-sheets, evaluating them via fragment programs [S1, unofficial]. More recently, with the rise of 3D Gaussian Splatting, LOD methods have become critical for handling large-scale scenes, leading to variants like Hierarchical 3DGS, Octree-GS, and SVL generation [S2, unofficial][S3, unofficial][S5, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] mkkellogg/GaussianSplats3D, https://github.com/mkkellogg/GaussianSplats3D, fetched 2026-10-27 (unofficial)
[S3] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S4] Differentiable Surface Rendering via Non-Differentiable Sampling, https://doi.org/10.1109/iccv48922.2021.00603, fetched 2026-10-26
[S5] wuyize25/gsplat-unity, https://github.com/wuyize25/gsplat-unity, fetched 2026-10-27 (unofficial)
[S6] DTIC ADA606342: An Anti-Aliasing Technique for Splatting, https://archive.org/details/DTIC_ADA606342, fetched 2026-10-26 (unofficial)
