# Visibility Culling


## Overview
Visibility culling encompasses techniques that determine which parts of a volume dataset contribute to the final image and bypass the computation for those that do not, thereby reducing the number of samples or primitives processed per frame [G]. Unlike space skipping, which bypasses empty regions along a ray, visibility culling typically addresses occlusion by other geometry or data, or uses geometric bounds to exclude entire datasets from the view [G]. It sets itself apart from its siblings by focusing on the *visibility* of data blocks or primitives relative to the camera and other visible surfaces, rather than just the emptiness of the space between the camera and the data.

## Shared principles
The core mechanism shared by all visibility culling methods is the reduction of work units (samples, voxels, or pixels) processed per frame while maintaining acceptable visual fidelity [G]. This is achieved through three primary strategies:
1. **Culling/Skipping:** Identifying regions of the volume or ray path that do not contribute to the final image (e.g., empty space, occluded voxels) and bypassing their evaluation [S2][S4, S5, unofficial].
2. **Termination:** Halting the integration loop when the accumulated contribution becomes negligible (e.g., high opacity, low transmittance) [S1, S6, unofficial].
3. **Adaptation:** Varying the computational effort based on local scene complexity or global performance budgets [S1, unofficial].

Mathematically, these methods modify the standard front-to-back compositing loop:
$$ C_{acc} = C_{acc} + C_{sample} \cdot \alpha_{sample} \cdot (1 - \alpha_{acc}) $$
$$ \alpha_{acc} = \alpha_{acc} + \alpha_{sample} \cdot (1 - \alpha_{acc}) $$
by introducing conditions that break the loop early, skip iterations, or reduce the number of $C_{sample}$ queries.

## Subtopics

### [Occlusion Clipping with Orthogonal Opacity Map](990-occlusion-clipping-with-orthogonal-opacity-map.md)
An acceleration technique for texture-based volume rendering that skips occluded voxels by creating a low-resolution opacity map on a plane orthogonal to a major dataset axis. It solves the problem of inefficient per-voxel occlusion testing in texture-based rendering by operating at the sub-volume level, culling fully hidden sub-volumes and clipping partially hidden ones. Unlike early ray termination which tests occlusion per-voxel, this method tests occlusion at the sub-volume level, which is more efficient for texture-based rendering where primitives contain thousands of voxels. It differs from hierarchical approaches like [Hidden Volume Removal using Hierarchical Occlusion Maps](2624-hidden-volume-removal-using-hierarchical-occlusion-maps.md) by using a fixed-resolution, multi-pass averaging strategy rather than a tree structure, and it complements [Acceleration](18-acceleration.md) by being more effective when there are many non-empty voxels and significant occlusion.

### [Early Termination and Probabilistic Path Control](1791-early-termination-and-probabilistic-path-control.md)
Early Termination and Probabilistic Path Control are visibility culling techniques that reduce computational cost by halting the ray-marching integration loop when the accumulated contribution to the final pixel becomes negligible. Unlike siblings such as [Empty-space skipping](1002-empty-space-skipping.md) which bypasses empty regions, or [Bounding Volume Culling](1239-bounding-volume-culling.md) which uses geometric bounds, this family terminates rays based on radiometric accumulation (opacity/transmittance) or statistical weight. It distinguishes itself from deterministic threshold methods by offering an unbiased alternative via probabilistic survival checks.

### [Probabilistic Culling](2833-probabilistic-culling.md)
Probabilistic culling is a rendering optimization that intentionally allows false-negatives in intersection tests to reduce duplicated fragments and rendering time, particularly in high-throughput scenarios like Virtual Reality (VR). It sets itself apart from conservative culling methods (such as its sibling [Occlusion Clipping with Orthogonal Opacity Map](990-occlusion-clipping-with-orthogonal-opacity-map.md)) by trading strict geometric accuracy for significant performance gains, justified by the fact that the resulting visual artifacts are virtually indistinguishable from correct renderings.

### [Visibility Feedback with Screen Space Subdivision](3439-visibility-feedback-with-screen-space-subdivision.md)
Visibility Feedback with Screen Space Subdivision is an out-of-core volume rendering optimization that uses visibility information gathered during the current frame to prevent the loading and refinement of occluded data in subsequent frames. Unlike standard early ray termination, which only reduces computation after data has already been loaded into the working set, this method integrates visibility feedback into the data loader to optimize GPU memory and bandwidth usage. It distinguishes itself from sibling methods by combining occlusion queries with a screen-space subdivision strategy to interleave query processing with rasterization, thereby reducing the latency inherent in asynchronous GPU occlusion queries.

### [Adaptive Multi-Step Empty Voxel Skipping](6289-adaptive-multi-step-empty-voxel-skipping.md)
Adaptive Multi-Step Empty Voxel Skipping is an acceleration technique for direct volume rendering that combines space-leaping (empty voxel skipping) with adaptive variable-step sampling to reduce redundant computation in sparse regions. Unlike traditional ray-casting which uses fixed equidistant sampling, this method detects empty regions to skip them entirely and adjusts the step length based on local voxel density, thereby lowering computational overhead while maintaining visual fidelity comparable to the original unaccelerated rendering. It builds on the principles of [Visibility Culling](6226-visibility-culling.md) by explicitly culling empty space and adapting computational effort.

### [Frustum Culling](8463-frustum-culling.md)
Frustum culling is a visibility culling technique that avoids rendering geometry located outside the camera's field of view, thereby eliminating unnecessary computation. It distinguishes itself from occlusion culling by relying solely on geometric containment relative to the viewing frustum planes, rather than depth-buffered visibility tests. It serves as a coarse-grained filter, often preceding more expensive fine-grained culling methods.

### Screen Regions
#### definition
- [S1, unofficial] An optimization dividing image space into equal-sized squares of n x n pixels to reduce overhead from per-pixel visibility testing. If a screen region is full, rasterization skips to the next boundary.
#### number
- [S1, unofficial] Screen regions save another 3 percent of overall rendering time on average.

### Visibility Culling via Scalar Range Lookup Table
#### definition
- [S2] A GPU-based visibility test using a 2*D* Boolean lookup table to cull cells that do not intersect the transfer function's opacity range, reducing load on the rendering stage.
#### step
- [S2] Geometry shader computes the minimum and maximum of the 8 cell corner values. These two values are used as table lookup indices to fetch a visibility value. If the cell is deemed visible, the geometry shader instantiates its 6 faces.
#### parameter
- [S2] Visibility Lookup Table: A 2*D* table where D is the number of scalar value bins. For each pair of values s1 and s2, the table holds 1 if there exists at least an opacity which is non-zero for scalar values in [s1, s2], otherwise 0.
#### number
- [S2] On a PC with two Xeon E5345 processors, 4GB memory, GeForce 8800 GTX (768MB), using the 'meteorite' dataset (2,377,878 cells), removing the look-up table drops performance to <1 fps (specifically 0.76 fps). Adding the look-up table increases framerate to ~3.48 fps.
- [S2] Adding the hybrid CPU-GPU mesh traversal mechanism to the look-up table optimization increases the framerate to 3.62 fps.

### 24-Region View-Dependent Visibility Preprocessing
#### definition
- [S3, unofficial] A preprocessing technique that decomposes the domain of viewing directions into 24 regions on a directional cube to extract a lower number of potentially visible voxels compared to the standard 6-region decomposition, improving extraction rate and rendering speed.
#### step
- [S3, unofficial] Decompose the viewing direction domain into 24 regions based on the signs of the x and y components, assuming the principal component of the viewing direction is the z coordinate.
- [S3, unofficial] Perform preprocessing for all 24 regions. During rendering, select the appropriate precomputed data structure based on the current viewing direction.
#### formula
- [S3, unofficial] The visibility function $v(i, j, k)$ for the case where x and y components are positive is defined as: $v(i, j, k) = 1$ if $k = 0$ or $\exists l, m \text{ s.t. } i - 1 \leq l \leq i, j - 1 \leq m \leq j, v(l, m, k - 1) = 1, b(l, m, k - 1) = 0$; otherwise 0. This checks only four voxels that could hide the current voxel.
#### number
- [S3, unofficial] On a low resolution dataset, decomposing into 24 regions extracts 1.8% of voxels, compared to 3.4% for 6 regions. On a high resolution volume, it extracts 0.9% compared to 1.8% for 6 regions.

### Interactive Render Quality Estimation & Cancellation
#### step
- [S4, unofficial] After octree traversal, check if the `OCTREE_RENDERING_CANCELED` flag is set in `_renderingRequired`.
- [S4, unofficial] If canceled, unlock the API, estimate a lower interactive render quality using `OCTREE_INTERACTIVE_RENDERING_POINT_THRESHOLD`, and return early to avoid a rendering loop.
#### parameter
- [S4, unofficial] `OCTREE_INTERACTIVE_RENDERING_POINT_THRESHOLD`: Threshold used to estimate a safe render quality when interactive traversal is canceled.
#### pitfall
- [S4, unofficial] Overestimating render quality can cause an infinite rendering loop. The cancellation check and quality estimation prevent this by forcing a lower quality on the next frame.

### Visible Zone Algorithm
#### definition
- [S5, unofficial] A conservative visibility computation algorithm with incremental updates, developed by Hall-Holt and Rusinkiewicz [58].
#### number
- [S5, unofficial] Able to achieve real-time frame rates for 2D and 2.5D datasets.

## Background
Visibility culling in volume rendering has evolved from simple geometric bounds to complex, view-dependent, and feedback-driven mechanisms. Early approaches focused on geometric culling, such as frustum culling, to remove datasets entirely outside the view [S2][S4, unofficial]. As datasets grew larger, techniques like the Visible Zone Algorithm were developed to provide conservative visibility with incremental updates, enabling real-time frame rates for large datasets [S5, unofficial].

The introduction of texture-based rendering and GPU acceleration led to methods like Occlusion Clipping with Orthogonal Opacity Maps, which leveraged the frame buffer to track occlusion at the sub-volume level [S1, unofficial]. Concurrently, probabilistic methods emerged to handle high-throughput scenarios like VR, trading strict accuracy for performance [S2]. More recent advancements integrate visibility feedback into out-of-core rendering pipelines, using screen-space subdivision to hide query latency and optimize memory bandwidth [S1, unofficial].

## Sources
[S1] (PDF) Cell-Based First-Hit Ray Casting - ResearchGate, https://www.researchgate.net/publication/2521319_Cell-Based_First-Hit_Ray_Casting, fetched 2026-10-27 (unofficial)
[S2] High-Quality, Semi-Analytical Volume Rendering for AMR ..., https://people.freedesktop.org/~marcheu/vis09-amr.pdf, fetched 2026-10-26
[S3] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S4] larsxschneider/iVoxel, https://github.com/larsxschneider/iVoxel, fetched 2026-10-27 (unofficial)
[S5] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
