# Hierarchical Raster Occlusion Culling

Also called: Lee et al. hierarchical raster occlusion culling

## Overview
Hierarchical Raster Occlusion Culling is a fully GPU-driven technique that reduces rasterization overhead by performing rasterization on a coarser level for group nodes, and refining visibility results through ray traversal down to the leaf level [S1]. It builds on [Raster Occlusion Culling](1562-raster-occlusion-culling.md) by applying spatial coherence exploitation through hierarchical grouping of objects [S1].

## How it works
**Inputs and Outputs:**
* Input: Hierarchical spatial structure of voxel blocks and camera parameters [G].
* Output: Dense list of visible leaf-level blocks for rendering [G].

**Data Structures:**
* Hierarchy Tree: Nodes are organized in levels, with group nodes representing aggregated bounding volumes of their children and leaf nodes representing individual voxel blocks [S1, G].
* Proxy Geometry: Each node has an associated proxy geometry (typically an axis-aligned bounding box) used for rasterization [G].

**Steps:**
1. **Coarse Rasterization:** Rasterize the proxy geometry of group nodes at a higher level of the hierarchy to determine which coarse regions potentially contain visible geometry [S1].
2. **Ray Traversal Refinement:** For regions identified as potentially visible in the coarse pass, perform ray traversal down the hierarchy tree by casting rays from the camera through the coarse visible regions to intersect child nodes [S1].
3. **Leaf Visibility Determination:** Continue traversal until leaf nodes are reached, marking a leaf block visible if the ray traversal confirms it is not occluded [S1].
4. **Stream Compaction:** Collect the indices of all visible leaf blocks into a dense list for rendering [G].

**Formulas:**
Not given in the sources.

**Parameters:**
* **Hierarchy Depth:** The number of levels to rasterize coarsely. Not given in the sources.
* **Ray Count:** Number of rays per coarse region. Not given in the sources.

**Edge Cases and Pitfalls:**
* **Precision:** Ray-triangle intersection precision at leaf levels may cause flickering if not handled with epsilon offsets [G].
* **Degenerate Input:** Empty regions in the hierarchy should be skipped during traversal to avoid unnecessary ray casting [G].

## Subtopics

### Iterative Raster Occlusion Culling
An iterative version of raster occlusion culling that works on a select number of hierarchy levels, applying spatial coherence exploitation through hierarchical grouping of objects [S1].

## Operating characteristics
* **Time Complexity:** Reduced compared to full leaf rasterization, as only coarse nodes are rasterized initially [S1]. Exact complexity depends on hierarchy depth and ray traversal cost. Not given in the sources.
* **Memory Complexity:** Requires storage for the hierarchy tree and proxy geometries. Not given in the sources.
* **Strengths:** Significantly reduces rasterization overhead by avoiding per-leaf rasterization and exploits spatial coherence [S1].
* **Limitations:** Ray traversal adds computational overhead. Not given in the sources.
* **When to use:** Large-scale volume rendering with high block counts where rasterization is the bottleneck. Not given in the sources.
* **When not to use:** Scenes with very few blocks or where ray traversal overhead exceeds rasterization savings. Not given in the sources.

## Background
The concept of hierarchical grouping for occlusion culling was introduced by Martin et al. [MBW08] [S1]. Lee and Lee [LL20] proposed an iterative version of raster occlusion culling [S1]. Lee et al. [LJSL21] introduced the fully GPU-driven hierarchical raster occlusion culling algorithm, aiming to reduce overall rasterization overhead by combining coarse rasterization with ray traversal refinement [S1].

## Sources
[S1] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
