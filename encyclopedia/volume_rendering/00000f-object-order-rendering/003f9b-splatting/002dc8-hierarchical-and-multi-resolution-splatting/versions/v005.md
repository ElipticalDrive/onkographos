# Hierarchical and Multi-Resolution Splatting

Also called: Multi-Resolution Splatting, Hierarchical Splatting, LOD Splatting

## Overview
Hierarchical and Multi-Resolution Splatting addresses the performance bottleneck of rendering high-density volumetric or point-cloud data by adaptively selecting the level of detail (LOD) based on screen-space footprint or error metrics. It sets itself apart from standard splatting by integrating spatial hierarchies (such as octrees or pyramids) to cull empty regions and aggregate primitives, allowing real-time interaction with datasets that would otherwise exceed memory or rasterization limits [S1, S2, unofficial][S4].

## Shared principles
The family shares the fundamental mechanism of decomposing a dense set of primitives into a spatial hierarchy where internal nodes store aggregated properties (mean position, covariance, opacity, or scalar value) of their children. The rendering process involves traversing this hierarchy and selecting a node to render as a single primitive if it satisfies a specific criterion, typically:
1.  **Screen-Space Footprint:** The projected size of the primitive is smaller than a threshold (e.g., a few pixels), making further subdivision visually insignificant [S3, unofficial].
2.  **Error Threshold:** The statistical error (e.g., RMS difference) between the node's aggregate and its children is below a user-defined precision bound [S1, unofficial][S2, unofficial].

When a node is selected, its aggregated properties are used to construct a single splat (polygon or Gaussian) which is then rasterized and composited. This reduces the number of draw calls and rasterization operations from $O(N)$ to $O(\sqrt{N})$ or better, depending on the data distribution.

## Subtopics

### [Hierarchical Splatting](1168-hierarchical-splatting.md)
A progressive refinement algorithm using a pyramidal volume representation and an adaptive octree. It constructs a hierarchy of coarser approximations by averaging $2 \times 2 \times 2$ blocks of samples. It sets itself apart by explicitly fitting an octree to the pyramid based on a user-supplied precision threshold, allowing the renderer to skip low-detail regions. Choose this for scalar volume data where error-bounded approximation is required.

### [Multi-Resolution Points](4784-multi-resolution-points.md)
A technique to render one large point in place of many small points to reduce rasterization cost. It solves the problem of high computational load in dense point clouds by adaptively selecting point sizes based on a local RMS error metric. Unlike siblings that may rely solely on screen-space footprint, this method explicitly controls point size by volume content. Choose this for unstructured point cloud rendering.

### [3D Gaussian Splatting LOD methods](5439-3d-gaussian-splatting-lod-methods.md)
Methods that address memory bottlenecks and rendering latency of standard 3D Gaussian Splatting (3DGS) by organizing primitives into hierarchical structures (e.g., Octree, SPT) or compressed representations. They set themselves apart by specifically adapting the Gaussian primitive's covariance and opacity properties for hierarchical aggregation (merging covariances and opacities) and preserving anisotropic nature during downsampling. Choose this for novel view synthesis of large-scale scenes using 3DGS.

### [Splat List (SVL) Generation with Multi-Level Downsampling](5767-splat-list-svl-generation-with-multi-level-downsampling.md)
A method that generates a Sparse Voxel List (SVL) of splat geometry across multiple LODs by iteratively downsampling a voxel occupancy mask. It differs from standard hierarchical splatting by explicitly constructing a contiguous buffer of `uint16_t` indices referencing geometry items, rather than relying on tree traversal during rasterization. Choose this for GPU-accelerated rendering pipelines requiring efficient buffer management.

### [Octree-based Splatting](8067-octree-based-splatting.md)
An object-order interactive volume rendering technique that accelerates rendering by approximating volume contributions using an octree hierarchy. It solves the performance bottleneck of dense voxel splatting by culling large empty or smooth regions. Unlike standard splatting which processes every voxel, this method selects the appropriate level of detail based on a stored error metric. Choose this for traditional scalar volume visualization.

### RBF Splatting with Octree and Fragment Programs
- **Definition:** A splatting implementation using Rational Basis Functions (RBFs) located by an octree, evaluated via fragment programs on cache-sheets [S1, unofficial].
- **Step:** Add an octree to locate the RBFs intersecting the cache-sheets. For each sheet plane pixel, use a fragment program to evaluate the exponential Gaussian function of the RBFs [S1, unofficial].
- **Performance:** 7 to 70 fps for Blunt-fin rendering at 400^2 resolution, caused by large variation of RBFs. Hardware: 2.8 GHz Pentium-IV, nVidia GeForce FX 5900 [S1, unofficial].

### Hierarchical Splat Culling
- **Definition:** Splats are culled prior to sorting and rendering using a custom octree to optimize performance [S2, unofficial].

### TreeSplat
- **Definition:** A method that organizes deformation as a mergeable tree, where each node represents a rigid part of an articulated object, and the overall deformation is computed by traversing the kinematic chain [S3, unofficial].

### Depth-Aware Multi-Layer Splatting
- **Definition:** A splatting strategy that converts rasterized surface points into Gaussian splats and uses a multi-layer accumulation strategy based on depth to handle occlusion boundaries and disocclusions, producing smooth derivatives [S4].
- **Step:** Render multiple layers of G-buffers. Maintain three accumulation buffers during splatting: $S^+$ for splats occluding the target pixel, $S^-$ for occluded splats, and $S^o$ for splats at the same depth as the target pixel. To determine splat depth relation, pair up multi-layer surface intersections at splat center $p$ with closest intersections at target pixel $q$ in depth. The $p$ layer paired with the front-most $q$ layer is assigned to $S^o$, layers in front of it to $S^+$, and the rest to $S^-$. After all splats are rendered, buffers are separately normalized following Eq. 3 and composited in $S^-$, $S^o$, $S^+$ order using over-compositing to produce the final result $S$ [S4].
- **Pitfall:** Single-layer splatting treats all splats at the same depth, ignoring occlusions and producing spurious derivatives for occluded objects. Multi-layer splatting corrects this by distinguishing front, back, and coincident depths [S4].

### Splat Downscaling Optimization
- **Definition:** Reduces Gaussian splat sizes to improve rendering speed while minimizing visual quality impact [S5, unofficial].
- **Parameter:** SplatDownscaleFactor: Float parameter in range [0, 1] controlling the degree of size reduction [S5, unofficial].

### Splatting-Based AMR Volume Rendering
- **Definition:** A volume rendering technique using kD-tree and octree-based domain subdivision to specify isovalue ranges and transfer functions, rendering via hierarchical splatting [S6, unofficial].
- **Step:** Subdivide the domain using kD-tree and octree structures. Specify isovalue range and transfer function. Render using hierarchical splatting [S6, unofficial].

## Background
Hierarchical splatting originated as a solution to the "fill-rate" and "overdraw" limitations of early GPU volume rendering. Early implementations, such as those by Cabral et al., demonstrated the potential of texture-based splatting [S1, unofficial]. The integration of octrees and pyramids allowed for adaptive resolution, enabling interactive frame rates for mid-sized datasets on hardware like the Pentium-IV and GeForce FX 5900 [S1, unofficial]. More recently, with the rise of 3D Gaussian Splatting, hierarchical methods have evolved to handle the massive memory footprint of large-scale scenes (e.g., city-scale), utilizing structures like Sequential Point Trees (SPT) and octrees to manage LODs dynamically [S2, S3, unofficial][S4].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] mkkellogg/GaussianSplats3D, https://github.com/mkkellogg/GaussianSplats3D, fetched 2026-10-27 (unofficial)
[S3] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S4] Differentiable Surface Rendering via Non-Differentiable Sampling, https://doi.org/10.1109/iccv48922.2021.00603, fetched 2026-10-26
[S5] wuyize25/gsplat-unity, https://github.com/wuyize25/gsplat-unity, fetched 2026-10-27 (unofficial)
[S6] Visualization Tools for Adaptive Mesh Refinement Data, https://www.vacet.org/publications/AMRVisJune2007.pdf, fetched 2026-10-27 (unofficial)
