# Visibility Culling

Also called: Visibility filtering

## Overview
Visibility culling reduces rendering load by rejecting geometry, chunks, or primitives that do not contribute to the final image, serving as a primary filter in image-order rendering pipelines to avoid unnecessary computation for objects not visible to the user [S2, S3, unofficial]. It differs from [Empty-space skipping](1002-empty-space-skipping.md) by operating on the accumulated radiometric state or geometric bounds rather than proactive spatial intervals, and from [Early Termination and Probabilistic Path Control](1791-early-termination-and-probabilistic-path-control.md) by excluding entire data structures or bounding volumes from the pipeline rather than stopping ray traversal [S2, S3, unofficial].

## Shared principles
The core mechanism shared by all methods in this family is the reduction of work units (samples, voxels, or pixels) processed per frame while maintaining acceptable visual fidelity [G]. This is achieved through three primary strategies:
1. **Culling/Skipping:** Identifying regions of the volume or ray path that do not contribute to the final image (e.g., empty space, occluded voxels) and bypassing their evaluation [S2][S4, S5].
2. **Termination:** Halting the integration loop when the accumulated contribution becomes negligible (e.g., high opacity, low transmittance) [S1, S6, unofficial].
3. **Adaptation:** Varying the computational effort based on local scene complexity or global performance budgets [S1, unofficial].

Mathematically, these methods modify the standard front-to-back compositing loop:
$$ C_{acc} = C_{acc} + C_{sample} \cdot \alpha_{sample} \cdot (1 - \alpha_{acc}) $$
$$ \alpha_{acc} = \alpha_{acc} + \alpha_{sample} \cdot (1 - \alpha_{acc}) $$
by introducing conditions that break the loop early, skip iterations, or reduce the number of $C_{sample}$ queries.

## Subtopics

### [Occlusion Clipping with Orthogonal Opacity Map](990-occlusion-clipping-with-orthogonal-opacity-map.md)
Occlusion Clipping with Orthogonal Opacity Map is an acceleration technique for texture-based volume rendering that skips occluded voxels by maintaining a low-resolution 2D opacity map on a plane orthogonal to a major dataset axis. It differs from per-voxel early ray termination by testing occlusion at the sub-volume level, allowing it to cull entire bounding boxes or clip their geometry against the opacity map to avoid rendering thousands of voxels hidden behind opaque surfaces.

### [Early Termination and Probabilistic Path Control](1791-early-termination-and-probabilistic-path-control.md)
Early Termination and Probabilistic Path Control is a family of volume rendering optimizations that reduce computational cost by halting the ray-marching integration loop or skipping samples when the remaining contribution to the pixel color is deemed negligible. It solves the problem of wasted computation on transparent regions or rays that have already accumulated sufficient opacity to saturate the pixel. Unlike [Empty-space skipping](1002-empty-space-skipping.md), which proactively skips spatial intervals of empty data, this family reacts to the accumulated radiometric state (opacity, transmittance, or uncertainty) of the ray path. It differs from [Bounding Volume Culling](1239-bounding-volume-culling.md) by operating on the internal integration state rather than geometric bounds, and from [Russian Roulette](1371-russian-roulette.md) by distinguishing between deterministic cutoffs (biased) and probabilistic survival (unbiased).

### [Probabilistic Culling](2833-probabilistic-culling.md)
Probabilistic culling is a rendering optimization that intentionally allows false-negatives in intersection tests to reduce duplicated fragments and rendering time, particularly in high-resolution or VR contexts. It sets itself apart from conservative culling methods (like [Occlusion Clipping with Orthogonal Opacity Map](990-occlusion-clipping-with-orthogonal-opacity-map.md)) by trading strict geometric accuracy for performance gains, justified by the visual indistinguishability of the resulting artifacts.

### [Visibility Feedback with Screen Space Subdivision](3439-visibility-feedback-with-screen-space-subdivision.md)
Visibility Feedback with Screen Space Subdivision is an out-of-core volume rendering optimization that uses visibility information gathered during the current frame to prevent the loading and refinement of occluded data in subsequent frames. Unlike standard early ray termination, which only skips computation after data has already been loaded into the GPU working set, this method feeds visibility results back to the loader to reduce GPU memory usage and bandwidth. It distinguishes itself from sibling methods by combining occlusion queries with a screen-space subdivision strategy to interleave query processing with rasterization, thereby hiding the latency inherent in asynchronous occlusion query results.

### [Adaptive Multi-Step Empty Voxel Skipping](6289-adaptive-multi-step-empty-voxel-skipping.md)
Adaptive Multi-Step Empty Voxel Skipping is an acceleration technique for direct volume rendering that combines space leaping (empty voxel skipping) with adaptive variable step sampling to reduce redundant computation in empty regions while maintaining image quality. It sets itself apart from traditional equidistant ray casting by dynamically adjusting the sampling interval based on local voxel density and explicitly culling rays that traverse only empty space, whereas siblings like [Occlusion Clipping with Orthogonal Opacity Map](990-occlusion-clipping-with-orthogonal-opacity-map.md) rely on precomputed opacity maps and [Early Termination and Probabilistic Path Control](1791-early-termination-and-probabilistic-path-control.md) focus on alpha-based termination rather than geometric space leaping.

### [24-Region View-Dependent Visibility Preprocessing](6608-24-region-view-dependent-visibility-preprocessing.md)
A preprocessing technique that decomposes the domain of viewing directions into 24 regions on a directional cube to extract a lower number of potentially visible voxels compared to the standard 6-region decomposition, improving extraction rate and rendering speed. It sets itself apart from [Frustum Culling](8463-frustum-culling.md) by operating on directional sign patterns rather than geometric bounds, and from [Occlusion Clipping with Orthogonal Opacity Map](990-occlusion-clipping-with-orthogonal-opacity-map.md) by using a static precomputed visibility mask based on local neighborhood checks rather than runtime opacity accumulation.

### [Frustum Culling](8463-frustum-culling.md)
Frustum culling is a visibility culling technique that reduces rendering load by rejecting geometry, chunks, or primitives that lie outside the camera's view frustum or beyond a specified proximity threshold. It serves as a primary filter in image-order rendering pipelines to avoid unnecessary computation for objects not visible to the user. Unlike [Occlusion Clipping with Orthogonal Opacity Map](990-occlusion-clipping-with-orthogonal-opacity-map.md) which handles hidden surface removal, or [Early Termination and Probabilistic Path Control](1791-early-termination-and-probabilistic-path-control.md) which stops ray traversal, frustum culling operates at a coarser level to exclude entire data structures or bounding volumes from the rendering pipeline entirely.

### [Interactive Render Quality Estimation & Cancellation](8895-interactive-render-quality-estimation-cancellation.md)
This method prevents infinite rendering loops in interactive volume rendering by detecting when traversal fails to converge and forcing a lower-quality fallback render. It sets apart from siblings like [Early Termination and Probabilistic Path Control](1791-early-termination-and-probabilistic-path-control.md) by acting as a safety guard for the traversal loop itself rather than optimizing the ray casting process, and from [Visibility Culling](6226-visibility-culling.md) by reacting to runtime failure states rather than precomputing visibility. It solves the problem of non-terminating or excessively long interactive sessions when the renderer cannot determine sufficient detail.

### [Visible Zone Algorithm](9086-visible-zone-algorithm.md)
The Visible Zone Algorithm is a conservative visibility computation method for volume rendering that utilizes incremental updates to determine which parts of a dataset are potentially visible from a given viewpoint. It sets itself apart from sibling methods like [Occlusion Clipping with Orthogonal Opacity Map](990-occlusion-clipping-with-orthogonal-opacity-map.md) by focusing on exact, conservative visibility guarantees rather than probabilistic approximations, ensuring no visible detail is incorrectly culled. This approach is particularly effective for 2D and 2.5D datasets where real-time frame rates are required without sacrificing correctness.

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

## Background
Visibility culling is a fundamental acceleration technique in computer graphics, particularly in volume rendering where data density is high. The concept of frustum culling has been standard since the early days of 3D graphics. More specialized techniques, such as opacity map-based culling and view-dependent preprocessing, have been developed to handle the specific challenges of volumetric data, such as transparency and complex internal structures. The use of lookup tables for cell-based culling represents a more recent optimization for adaptive mesh refinement (AMR) data.

## Sources
[S1] (PDF) Cell-Based First-Hit Ray Casting - ResearchGate, https://www.researchgate.net/publication/2521319_Cell-Based_First-Hit_Ray_Casting, fetched 2026-10-27 (unofficial)
[S2] High-Quality, Semi-Analytical Volume Rendering for AMR ..., https://people.freedesktop.org/~marcheu/vis09-amr.pdf, fetched 2026-10-26
