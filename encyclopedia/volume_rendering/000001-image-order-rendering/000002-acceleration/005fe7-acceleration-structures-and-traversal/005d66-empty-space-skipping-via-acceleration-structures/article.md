# Empty Space Skipping via Acceleration Structures

Also called: Acceleration Structure-Based Empty Space Skipping, Hierarchical Skipping

## Overview
Empty space skipping via acceleration structures accelerates volume rendering by identifying and bypassing transparent or empty regions of the volume data before expensive sampling and compositing occur . Unlike sibling methods that may rely on geometric proxies [16346-progressive-ray-casting-for-volumetric-models-on-mobile-devi.md] or orthogonal opacity maps [990-occlusion-clipping-with-orthogonal-opacity-map.md], this family constructs explicit hierarchical or grid-based metadata structures (e.g., octrees, BVHs, min/max grids) to determine ray visibility [13062-empty-space-skipping-via-acceleration-structure.md]. It sets itself apart from [SparseLeap](5152-sparseleap.md) by typically using scalar min/max values or occupancy flags rather than rasterized segment lists, and from [Hierarchical Bounding Box Tree for Empty-Space Skipping](8182-hierarchical-bounding-box-tree-for-empty-space-skipping.md) by employing fixed-grid brick structures or specific tree traversals rather than generic bounding box hierarchies [13062-empty-space-skipping-via-acceleration-structure.md].

## Shared principles
The core mechanism shared by all children is the reduction of work units (samples, voxels, or pixels) processed per frame while maintaining acceptable visual fidelity [G]. This is achieved through three primary strategies:
1. **Culling/Skipping:** Identifying regions of the volume or ray path that do not contribute to the final image (e.g., empty space, occluded voxels) and bypassing their evaluation .
2. **Termination:** Halting the integration loop when the accumulated contribution becomes negligible (e.g., high opacity, low transmittance) [S1, unofficial].
3. **Adaptation:** Varying the computational effort based on local scene complexity or global performance budgets [S1, unofficial].

Mathematically, these methods modify the standard front-to-back compositing loop:
$$ C_{acc} = C_{acc} + C_{sample} \cdot \alpha_{sample} \cdot (1 - \alpha_{acc}) $$
$$ \alpha_{acc} = \alpha_{acc} + \alpha_{sample} \cdot (1 - \alpha_{acc}) $$
by introducing conditions that break the loop early, skip iterations, or reduce the number of $C_{sample}$ queries.

## Subtopics

### [SparseLeap](5152-sparseleap.md)
SparseLeap is a hybrid object-order (rasterization) and image-order (ray-casting) method for efficient empty space skipping in large, sparse volumes with fine structures. It solves the problem of high computational cost and space fragmentation inherent in standard hierarchical empty space skipping (e.g., octrees) by decoupling the empty space skipping data structures from the underlying volume representation. Unlike standard octree methods that suffer from fragmentation and high traversal costs, SparseLeap rasterizes view-independent occupancy geometry to generate per-pixel linked lists of ray segments, enabling "as-long-as-possible" skipping via simple linear list traversal without hierarchy traversal.

### [Hierarchical Bounding Box Tree for Empty-Space Skipping](8182-hierarchical-bounding-box-tree-for-empty-space-skipping.md)
This method accelerates traditional ray casting for large-scale aerosol visualization (smoke, fog, dust) by combining a Hierarchical Bounding Box Tree (BVH) for empty voxel skipping, early termination of light compositing, and grouped linear interpolation to reduce computational load. It sets itself apart from sibling methods like [SparseLeap](5152-sparseleap.md) and [Efficient Space Skipping & Adaptive Sampling of Unstructured Volumes Using H.W. Accel. Ray Tracing](8217-efficient-space-skipping-adaptive-sampling-of-unstructured-v.md) by specifically employing a BVH constructed from an octree traversal to avoid the overhead of recording empty nodes, which increases complexity in pure octree-based approaches, while offering dynamic update capabilities superior to K-D trees.

### [SparseLeap Ray-Casting with Occupancy Misses](9818-sparseleap-ray-casting-with-occupancy-misses.md)
SparseLeap is a ray-guided volume rendering technique that separates meta-data for empty space skipping from the volume data itself, using a ray segment list to process rays. It extends standard ray-casting by introducing an outer loop over ray segments and handling 'occupancy misses' to delay the determination of whether a brick is empty or non-empty until it is actually needed. Unlike octree-based skipping which incurs high depth complexity due to fragmented space subdivision, SparseLeap limits depth complexity by generating fewer, longer ray segments.

### [Space Skipping and Empty Volume Culling](10063-space-skipping-and-empty-volume-culling.md)
Space skipping and empty volume culling are acceleration techniques in image-order volume rendering that reduce computational load by identifying and bypassing regions of the volume or ray path that do not contribute to the final image. What sets this family apart from its siblings is its primary focus on geometric or scalar-based determination of "visible" vs. "invisible" primitives (cells, voxels, or rays) before expensive integration occurs, whereas methods like [Early Termination and Probabilistic Path Control](1791-early-termination-and-probabilistic-path-control.md) operate during the ray-marching process itself [G].

### [Empty-space skipping via Acceleration Structure](13062-empty-space-skipping-via-acceleration-structure.md)
This method accelerates direct volume rendering by using a hierarchical min/max acceleration structure to skip transparent regions of the volume data. It reduces the number of ray samples taken by allowing rays to jump over blocks that do not contain visible data, typically achieving a 1.5–2x speedup. It differs from sibling methods like [Hierarchical Bounding Box Tree for Empty-Space Skipping](8182-hierarchical-bounding-box-tree-for-empty-space-skipping.md) by using a fixed-grid brick structure rather than a tree of bounding boxes, and from [SparseLeap](5152-sparseleap.md) by relying on scalar min/max values rather than occupancy or sparse data structures.

### [Progressive Ray Casting for Volumetric Models on Mobile Devices](16346-progressive-ray-casting-for-volumetric-models-on-mobile-devi.md)
Empty Space Skipping (ESS): Rays start sampling at the boundaries of a proxy geometry that roughly adjusts to non-transparent regions of the volume model, discarding transparent regions.

### Auxiliary terms for empty space skipping and smoke mitigation
- **Definition:** A method to skip empty space using a hierarchical formulation to reduce computational impact, and a quadratic term on visibility to mitigate smoke-like artifacts in real scenes [S1, unofficial].
- **Step:** For each hierarchical level above the first, store the voxel grid of the previous level. If the density value for a voxel in that grid is smaller than a threshold $\lambda_v = 1.0$, skip the voxel and treat it as empty [S1, unofficial].
- **Formula:** Auxiliary loss $L_s = \lambda_s (-4(T - 0.5)^2 + 1)$, where $T$ is the total visibility for each ray, and $\lambda_s = 0.1$. This term helps mitigate smoke-like artifacts and keeps real scenes sparse [S1, unofficial].
- **Parameter:** $\lambda_s = 0.1$ [S1, unofficial].

## Background
The concept of empty space skipping is fundamental to efficient volume rendering, particularly for large-scale datasets where most of the volume is transparent or empty. Early implementations relied on simple grid-based checks or octrees [13062-empty-space-skipping-via-acceleration-structure.md]. The use of hierarchical bounding box trees (BVH) for aerosol visualization was developed to handle large-scale smoke and fog [8182-hierarchical-bounding-box-tree-for-empty-space-skipping.md]. SparseLeap was introduced to address the fragmentation issues in standard octree-based skipping by decoupling the skipping data from the volume representation [5152-sparseleap.md]. The auxiliary terms for smoke mitigation were introduced in the context of view-dependent radiance fields to address shape-radiance ambiguity [S1, unofficial].

## Sources
[S1] Addressing the Shape-Radiance Ambiguity in View-Dependent Radiance Fields, https://doi.org/10.48550/arxiv.2203.01553, fetched 2026-10-29 (unofficial)
