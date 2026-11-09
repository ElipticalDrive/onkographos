# Hierarchical and Multi-Resolution Splatting

Also called: Multi-Resolution Splatting, Hierarchical Splatting, LOD Splatting

## Overview
Hierarchical and Multi-Resolution Splatting addresses the performance bottleneck of rendering high-density volumetric or point-cloud data by adaptively selecting the resolution of geometric primitives based on visual significance or screen-space footprint [S1, unofficial][S2, unofficial]. It sets itself apart from standard splatting by integrating spatial hierarchies (such as octrees or pyramids) or multi-level downsampling to cull low-detail regions, thereby reducing rasterization cost while maintaining image quality within a user-defined error bound [S1, unofficial][S2, unofficial].

## Shared principles
The family shares the fundamental mechanism of decomposing a dense dataset into a hierarchy of representations, where coarser levels approximate finer levels within a specified error tolerance.

1.  **Hierarchical Representation:** The input data (voxels, points, or Gaussians) is organized into a tree structure (e.g., octree, quadtree, or pyramid) [S1, unofficial][S2, unofficial]. Internal nodes store aggregate properties (mean position, covariance, total opacity, or average scalar value) of their children [S1, unofficial][S2, unofficial].
2.  **Error Estimation:** Each node computes an error metric comparing its aggregate properties to those of its children. Common metrics include Root Mean Square (RMS) error for scalar values or screen-space footprint size for Gaussian splats [S1, unofficial][S3, unofficial].
    $$ E = \sqrt{\frac{\sum w_i (C_i - A)^2}{\sum w_i}} $$
    where $A$ is the aggregate value, $C_i$ are children values, and $w_i$ are weights [S1, unofficial].
3.  **Adaptive Traversal:** During rendering, the hierarchy is traversed from root to leaves. A node is selected as a leaf (and thus rendered as a single primitive) if its error $E$ is below a threshold $\epsilon$ or if its projected screen-space size is below a pixel threshold [S1, unofficial][S3, unofficial]. Otherwise, the algorithm recurses to its children [S1, unofficial][S3, unofficial].
4.  **Primitive Aggregation:** When a node is selected, its children are merged into a single representative primitive. For Gaussians, this involves merging covariance matrices and opacities [S4]. For voxels, it involves averaging scalar values [S2, unofficial].
5.  **Rasterization:** The selected primitives are rasterized using standard splatting techniques, typically in back-to-front order to ensure correct alpha compositing [S1, unofficial][S2, unofficial].

## Subtopics

### [Hierarchical Splatting](1168-hierarchical-splatting.md)
Hierarchical splatting is a progressive refinement algorithm for volume rendering that uses a pyramidal volume representation to achieve real-time interactive rendering. It solves the problem of rendering high-resolution volumetric data efficiently by constructing a hierarchy of coarser approximations, allowing the renderer to select the appropriate resolution level based on a user-supplied precision threshold or desired image quality. It sets itself apart from standard splatting by integrating an adaptive octtree structure derived from the pyramid, enabling the system to skip low-detail regions and focus computational resources on areas requiring higher fidelity.

### [Multi-Resolution Points](4784-multi-resolution-points.md)
Multi-Resolution Points is a technique to render one large point in place of many small points to reduce rasterization cost and storage. It solves the problem of high fill-rate requirements and memory overhead when rendering dense point clouds by adaptively selecting a coarser representation for clusters of points that are visually insignificant. It sets itself apart from siblings by explicitly using a local error metric (such as RMS error) to control the granularity of the point hierarchy, rather than relying solely on screen-space footprint or fixed LOD levels.

### [3D Gaussian Splatting LOD methods](5439-3d-gaussian-splatting-lod-methods.md)
3D Gaussian Splatting (3DGS) Level-of-Detail (LOD) methods address the prohibitive memory consumption and rendering latency of monolithic 3D Gaussian representations, particularly in large-scale or high-fidelity scenes. While standard 3DGS requires storing and rasterizing all primitives, LOD methods decompose the scene into hierarchical structures or compressed representations to dynamically select the appropriate granularity based on camera distance, resolution, or hardware constraints. These methods distinguish themselves from sibling techniques like [Hierarchical Splatting](1168-hierarchical-splatting.md) by specifically adapting the 3DGS primitive (Gaussian) properties—such as covariance, opacity, and spherical harmonics—into multi-resolution hierarchies or quantized codebooks, rather than relying solely on generic point-cloud or voxel-based aggregation.

### [Splat List (SVL) Generation with Multi-Level Downsampling](5767-splat-list-svl-generation-with-multi-level-downsampling.md)
Splat List (SVL) Generation with Multi-Level Downsampling is a rendering technique that generates a sparse voxel list (SVL) of splat geometry across multiple levels of detail (LOD) by iteratively downsampling a voxel occupancy mask and extracting splat primitives for each level. It differs from standard hierarchical splatting by explicitly constructing a multi-resolution hierarchy through iterative mask downsampling rather than traversing a pre-existing spatial tree, allowing for efficient generation of LOD splats directly from voxel data.

### [Octree-based Splatting](8067-octree-based-splatting.md)
Octree-based Splatting is an object-order interactive volume rendering technique that approximates volume contributions using an octree representation to balance rendering speed and image quality. It solves the performance bottleneck of dense voxel rendering by culling voxels and aggregating data, allowing the renderer to descend the octree only until a node's error tolerance is met. Unlike standard splatting which processes every voxel, this method adapts the resolution of the splats based on the spatial hierarchy, rendering larger geometric primitives for coarse, uniform regions and finer primitives for detailed areas.

### RBF Splatting with Octree and Fragment Programs
- **Definition:** A splatting implementation using Rational Basis Functions (RBFs) located by an octree, evaluated via fragment programs on cache-sheets [S1, unofficial].
- **Steps:**
    1. Add an octree to locate the Rational Basis Functions (RBFs) intersecting the cache-sheets [S1, unofficial].
    2. For each sheet plane pixel, use a fragment program to evaluate the exponential Gaussian function of the RBFs [S1, unofficial].
- **Performance:** 7 to 70 fps for Blunt-fin rendering at $400^2$ resolution, caused by large variation of RBFs. Hardware: 2.8 GHz Pentium-IV, nVidia GeForce FX 5900 [S1, unofficial].

### Hierarchical Splat Culling
- **Definition:** Splats are culled prior to sorting and rendering using a custom octree to optimize performance [S2, unofficial].

### TreeSplat
- **Definition:** A method that organizes deformation as a mergeable tree, where each node represents a rigid part of an articulated object, and the overall deformation is computed by traversing the kinematic chain [S3, unofficial].

### Depth-Aware Multi-Layer Splatting
- **Definition:** A splatting strategy that converts rasterized surface points into Gaussian splats and uses a multi-layer accumulation strategy based on depth to handle occlusion boundaries and disocclusions, producing smooth derivatives [S4].
- **Steps:**
    1. Render multiple layers of G-buffers. Maintain three accumulation buffers during splatting: $S^+$ for splats occluding the target pixel, $S^-$ for occluded splats, and $S^o$ for splats at the same depth as the target pixel [S4].
    2. To determine splat depth relation, pair up multi-layer surface intersections at splat center $p$ with closest intersections at target pixel $q$ in depth. The $p$ layer paired with the front-most $q$ layer is assigned to $S^o$, layers in front of it to $S^+$, and the rest to $S^-$ [S4].
    3. After all splats are rendered, buffers are separately normalized following Eq. 3 and composited in $S^-$, $S^o$, $S^+$ order using over-compositing to produce the final result $S$ [S4].
- **Pitfall:** Single-layer splatting treats all splats at the same depth, ignoring occlusions and producing spurious derivatives for occluded objects. Multi-layer splatting corrects this by distinguishing front, back, and coincident depths [S4].

### Splat Downscaling Optimization
- **Definition:** Reduces Gaussian splat sizes to improve rendering speed while minimizing visual quality impact [S5, unofficial].
- **Parameter:** SplatDownscaleFactor: Float parameter in range $[0, 1]$ controlling the degree of size reduction [S5, unofficial].

### Splatting-Based AMR Volume Rendering
- **Definition:** A volume rendering technique using kD-tree and octree-based domain subdivision to specify isovalue ranges and transfer functions, rendering via hierarchical splatting [S6, unofficial].
- **Steps:**
    1. Subdivide the domain using kD-tree and octree structures [S6, unofficial].
    2. Specify isovalue range and transfer function [S6, unofficial].
    3. Render using hierarchical splatting [S6, unofficial].

## Background
Hierarchical splatting originated in the mid-1990s as a method to accelerate volume rendering by reducing the number of primitives processed. Early implementations used pyramidal volume representations and octrees to adaptively refine the rendering resolution based on error bounds [S1, unofficial][S2, unofficial]. The technique has since evolved to support various data types, including point clouds [S1, unofficial], 3D Gaussian Splatting [S3, unofficial], and Adaptive Mesh Refinement (AMR) data [S6, unofficial]. Recent developments focus on optimizing 3DGS LODs through quantization, pruning, and virtual memory management to handle large-scale scenes [S3, unofficial][S5, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] mkkellogg/GaussianSplats3D, https://github.com/mkkellogg/GaussianSplats3D, fetched 2026-10-27 (unofficial)
[S3] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S4] Differentiable Surface Rendering via Non-Differentiable Sampling, https://doi.org/10.1109/iccv48922.2021.00603, fetched 2026-10-26
[S5] wuyize25/gsplat-unity, https://github.com/wuyize25/gsplat-unity, fetched 2026-10-27 (unofficial)
[S6] Visualization Tools for Adaptive Mesh Refinement Data, https://www.vacet.org/publications/AMRVisJune2007.pdf, fetched 2026-10-27 (unofficial)
