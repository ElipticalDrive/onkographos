# Acceleration Structures and Traversal

Also called: Volume Acceleration, Ray Traversal

## Overview
This family encompasses the algorithmic strategies for navigating volumetric data structures and the spatial metadata used to skip empty or occluded regions during ray casting [13739-traversal-and-data-access.md][17387-empty-space-skipping-via-acceleration-structures.md]. It solves the problem of mapping continuous ray paths to discrete data elements while minimizing memory bandwidth and computation [13739-traversal-and-data-access.md]. It sets itself apart from sibling methods like [GL-NeRF](803-gl-nerf.md) and [Occlusion Clipping with Orthogonal Opacity Map](990-occlusion-clipping-with-orthogonal-opacity-map.md) by focusing on the topology of the data access path and explicit hierarchical culling rather than neural approximation or rasterized proxies [13739-traversal-and-data-access.md][17387-empty-space-skipping-via-acceleration-structures.md].

## Shared principles
The core mechanism shared by all children is the determination of the sequence of data elements (voxels, cells, or bricks) intersected by a ray, and the retrieval of values from those elements [G]. This involves:
1. **Path Parameterization:** Representing the ray as $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ and computing intersection intervals $[t_{entry}, t_{exit}]$ with data boundaries [G].
2. **Coordinate Mapping:** Transforming the continuous ray parameter $t$ into discrete data indices (e.g., voxel coordinates, cell IDs, or brick IDs) [G].
3. **Data Retrieval:** Fetching the value at the mapped index from the appropriate memory structure (texture, buffer, or mesh) [G].

Mathematically, for a uniform grid, this is often a Digital Differential Analyzer (DDA) step. For irregular grids, it involves explicit boundary intersection tests. For hierarchical data, it involves Level of Detail (LoD) selection and cache residency checks [G].

## Subtopics

### [Traversal and Data Access](13739-traversal-and-data-access.md)
Covers the algorithmic strategies for moving a ray through a volume dataset and retrieving scalar values for compositing [13739-traversal-and-data-access.md]. It sets itself apart from siblings like [GL-NeRF](803-gl-nerf.md) and [Space Skipping and Empty Volume Culling](10063-space-skipping-and-empty-volume-culling.md) by focusing on the topology of the data access path and memory availability [13739-traversal-and-data-access.md].

### [LLO-based Ray-casting](16481-llo-based-ray-casting.md)
A volume rendering technique using a Linear Level Octree (LLO) spatial data structure to accelerate ray-casting by skipping empty space and utilizing early ray termination [16481-llo-based-ray-casting.md].

### [Empty Space Skipping via Acceleration Structures](17387-empty-space-skipping-via-acceleration-structures.md)
Accelerates volume rendering by identifying and bypassing transparent or empty regions of the volume data before expensive sampling and compositing occur [17387-empty-space-skipping-via-acceleration-structures.md]. Unlike sibling methods that may rely on geometric proxies [16346-progressive-ray-casting-for-volumetric-models-on-mobile-devi.md] or orthogonal opacity maps [990-occlusion-clipping-with-orthogonal-opacity-map.md], this family constructs explicit hierarchical or grid-based metadata structures (e.g., octrees, BVHs, min/max grids) to determine ray visibility [13062-empty-space-skipping-via-acceleration-structure.md]. It sets itself apart from [SparseLeap](5152-sparseleap.md) by typically using scalar min/max values or occupancy flags rather than rasterized segment lists, and from [Hierarchical Bounding Box Tree for Empty-Space Skipping](8182-hierarchical-bounding-box-tree-for-empty-space-skipping.md) by employing fixed-grid brick structures or specific tree traversals rather than generic bounding box hierarchies [13062-empty-space-skipping-via-acceleration-structure.md].

## Background
Not given in the sources.

## Sources
