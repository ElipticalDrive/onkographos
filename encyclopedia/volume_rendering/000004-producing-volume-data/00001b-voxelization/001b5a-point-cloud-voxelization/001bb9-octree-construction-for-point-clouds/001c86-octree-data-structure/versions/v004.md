# Octree Data Structure

Also called: CGAL::Octree, CGAL::Orthtree, RFVoxelOctree, RFVoxelOctreeNode, Voxel Octree Data Structure

## Overview
The Octree Data Structure partitions 3D space into a hierarchical grid of voxels to organize point cloud objects for rendering [S1, unofficial][S2]. It recursively subdivides a bounding volume into eight octants until a leaf condition is met, where leaf nodes represent occupied voxels or empty space with surface distance metrics [S1, unofficial]. It differs from its siblings by providing the foundational spatial indexing for voxelization and isosurfacing, whereas [Octree Construction for Point Clouds](1070-octree-construction-for-point-clouds.md) describes the build process and [Octree-based Level of Detail and Culling](1079-octree-based-level-of-detail-and-culling.md) describes traversal optimizations.

## How it works
**Inputs and Outputs:**
*   **Input:** A point cloud or implicit function domain, defined by a bounding box (e.g., $[-1, 1]^3$) and a voxel resolution parameter [S2].
*   **Output:** A tree structure where internal nodes branch into 8 children, and leaf nodes store occupancy status and geometric data (e.g., surface distance) [S1, unofficial][S2].

**Data Structures:**
Two primary implementations are cited:

1.  **RFVoxelOctree (Custom Implementation):**
    *   `RFVoxelOctreeNode` fields [S1, unofficial]:
        *   `m_pkChildren[8]`: Array of 8 pointers to child nodes.
        *   `m_kBuildAabb`: `RFAABB3i` (integer axis-aligned bounding box).
        *   `m_kWorldAabb`: `RFAABB3d` (double axis-aligned bounding box).
        *   `m_eNodeType`: Enum (`NT_INTERNAL`, `NT_EMPTY_LEAF`, `NT_LEAF`).
        *   `m_fDistanceToSurface`: `double` (surface distance metric for empty leaves).
    *   `RFVoxelOctree` fields [S1, unofficial]:
        *   `m_pkRoot`: Pointer to root node.
        *   `m_kDataDimension`: `RFVector3i`.
        *   `m_kOctreeOrigin`: `RFVector3d`.
        *   `m_fVoxelSize`: `double` (physical size of a single voxel unit, default 1.0f).
        *   `m_uiOctreeDim`: `unsigned int` (total dimension of the cubic voxel grid).
        *   `m_uiOctreeDepth`: `unsigned int` (derived as $\log_2(m\_uiOctreeDim)$).
        *   `m_uiNumberOfNodes`, `m_uiNumberOfLeafs`, `m_uiNumberOfEmptyLeafs`, `m_uiNumberOfInternalNodes`: Counters.

2.  **CGAL::Orthtree (CGAL Library):**
    *   Defined as `CGAL::Orthtree< Orthtree_traits_point< GeomTraits, PointRange, PointMap, cubic_nodes, 3 > >` [S2].
    *   Supports `CGAL::Orthtrees::Leaves_traversal` for iterating over leaf nodes [S2].
    *   Supports `dump_to_polylines` for exporting structure [S2].

**Algorithm Steps:**
1.  **Initialization:** Define the global bounding box and initial voxel size/dimensions [S1, unofficial][S2].
2.  **Subdivision:** Recursively divide the current node's bounding volume into 8 equal octants [S1, unofficial].
3.  **Leaf Condition:** Stop subdivision when a node meets the leaf criteria (e.g., contains a single point, or depth limit reached) [S1, unofficial][S2].
4.  **Storage:**
    *   If `NT_LEAF`: Store point data or surface distance metric [S1, unofficial].
    *   If `NT_EMPTY_LEAF`: Store surface distance metric for potential extraction [S1, unofficial].
    *   If `NT_INTERNAL`: Initialize 8 child pointers [S1, unofficial].
5.  **Traversal:** Use `Leaves_traversal` to access leaf nodes for rendering or meshing [S2].

**Parameters:**
*   `m_fVoxelSize`: Physical size of a voxel. Default: 1.0f [S1, unofficial].
*   `m_uiOctreeDim`: Total grid dimension. Depth = $\log_2(m\_uiOctreeDim)$ [S1, unofficial].
*   `isovalue`: Threshold for surface extraction (e.g., 0.3) [S2].

**Edge Cases and Pitfalls:**
*   **Cracks:** Varying depths around an isosurface produce cracks at boundaries where octree levels are discontinuous [S2].
*   **Uniformity:** The structure behaves well if nodes are split with uniform size around the surface; non-uniform refinement requires careful handling to maintain manifold surfaces [S2].

## Subtopics

### [Structured Point Cloud Octree](13087-structured-point-cloud-octree.md)
Encodes a sparse octree in a compact, batch-friendly tensor format optimized for GPU execution, specifically designed for high-throughput queries in neural radiance fields (NeRF).

## Operating characteristics
*   **Time Complexity:** Construction depends on the number of points and depth $D = \log_2(\text{Dim})$. Traversal is proportional to the number of leaves.
*   **Memory Complexity:** Proportional to the number of nodes. Each internal node stores 8 pointers and bounding boxes [S1, unofficial].
*   **Strengths:** Hierarchical culling; efficient spatial queries; supports surface distance metrics for empty space [S1, unofficial].
*   **Limitations:** Non-uniform refinement can cause mesh cracks during isosurfacing [S2].
*   **When to use:** When organizing point clouds for voxelization or accelerating ray marching in neural fields [S1, unofficial][S2].
*   **When not to use:** When uniform grid performance is sufficient and overhead of tree structure is unjustified.

## Background
The CGAL implementation originated from the 2022 Google Summer of Code, contributed by Julian Stahl, mentored by Daniel Zint and Pierre Alliez, providing initial Marching Cubes, Topologically Correct Marching Cubes, and Dual Contouring support [S2]. The RFVoxelOctree structure is part of the Voxelizer project [S1, unofficial].

## Sources
[S1] itsyouyi/Voxelizer, https://github.com/itsyouyi/Voxelizer, fetched 2026-10-27 (unofficial)
[S2] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
