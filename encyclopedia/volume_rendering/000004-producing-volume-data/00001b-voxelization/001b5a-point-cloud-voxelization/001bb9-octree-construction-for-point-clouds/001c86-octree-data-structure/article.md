# Octree Data Structure

Also called: CGAL::Octree, CGAL::Orthtree, RFVoxelOctree, RFVoxelOctreeNode, Voxel Octree Data Structure

## Overview
The octree partitions 3D space into a hierarchical grid of voxels to organize point cloud data for volume rendering and surface extraction [S1, unofficial][S2]. It recursively subdivides space into eight octants, distinguishing it from flat voxel grids [S1, unofficial].

## How it works
**Inputs:** A set of 3D points (point cloud) and a bounding volume defining the spatial extent [S2].
**Outputs:** A tree structure where internal nodes represent partially filled space, leaf nodes represent occupied voxels, and empty leaf nodes retain a surface distance metric [S1, unofficial].

### Data Structures
Two primary implementations are cited: the RFVoxelOctree [S1, unofficial] and CGAL::Orthtree [S2].

**RFVoxelOctreeNode Layout [S1, unofficial]:**
*   `m_pkChildren[8]`: Array of 8 pointers to child nodes.
*   `m_kBuildAabb`: Axis-aligned bounding box in build space.
*   `m_kWorldAabb`: Axis-aligned bounding box in world space.
*   `m_eNodeType`: Enum indicating state: `NT_INTERNAL`, `NT_EMPTY_LEAF`, or `NT_LEAF`.
*   `m_fDistanceToSurface`: Double-precision distance to the nearest surface (for empty leaves).

**RFVoxelOctree Layout [S1, unofficial]:**
*   `m_pkRoot`: Pointer to the root node.
*   `m_kDataDimension`: 3D integer vector defining the grid dimensions.
*   `m_kOctreeOrigin`: 3D double vector for the octree origin.
*   `m_fVoxelSize`: Physical size of a single voxel unit (default 1.0f).
*   `m_uiOctreeDim`: Total dimension of the cubic voxel grid. Depth is derived as $\log_2(m\_uiOctreeDim)$ [S1, unofficial].
*   `m_uiOctreeDepth`: Calculated depth.
*   `m_uiNumberOfNodes`, `m_uiNumberOfLeafs`, `m_uiNumberOfEmptyLeafs`, `m_uiNumberOfInternalNodes`: Counters for statistics [S1, unofficial].

**CGAL::Orthtree Definition [S2]:**
Defined as `CGAL::Orthtree< Orthtree_traits_point< GeomTraits, PointRange, PointMap, cubic_nodes, 3 > >` [S2]. This template specialization uses cubic nodes in 3 dimensions [S2].

### Algorithm Steps
1.  **Initialization:** Define the bounding box and point range [S2]. Set `m_fVoxelSize` (default 1.0f) and `m_uiOctreeDim` [S1, unofficial].
2.  **Construction:** Recursively subdivide the bounding volume into 8 octants [S1, unofficial]. CGAL supports specific refinement splitters, such as `Refine_one_eighth` or `Refine_around_isovalue`, to control refinement density around the isosurface [S2].
3.  **Leaf Classification:** Leaf nodes represent occupied voxels [S1, unofficial]. Empty leaf nodes represent empty space but retain `m_fDistanceToSurface` for potential surface extraction [S1, unofficial].
4.  **Traversal:** Supports iteration over leaf nodes via `CGAL::Orthtrees::Leaves_traversal` [S2].
5.  **Output:** Nodes can be dumped to polylines via `octree.dump_to_polylines(oo)` or used for Marching Cubes (MC), Topologically Correct Marching Cubes (TMC), or Dual Contouring (DC) [S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
from math import log2
from enum import Enum

class NodeType(Enum):
    NT_INTERNAL = 0
    NT_EMPTY_LEAF = 1
    NT_LEAF = 2

class RFAABB3i:
    def __init__(self, min_val, max_val):
        self.min = min_val
        self.max = max_val

class RFAABB3d:
    def __init__(self, min_val, max_val):
        self.min = min_val
        self.max = max_val

class RFVector3i:
    def __init__(self, x, y, z):
        self.x, self.y, self.z = x, y, z

class RFVector3d:
    def __init__(self, x, y, z):
        self.x, self.y, self.z = x, y, z

class RFVoxelOctreeNode:
    def __init__(self):
        self.m_pkChildren = [None] * 8
        self.m_kBuildAabb = RFAABB3i((0, 0, 0), (1, 1, 1))
        self.m_kWorldAabb = RFAABB3d((0.0, 0.0, 0.0), (1.0, 1.0, 1.0))
        self.m_eNodeType = NodeType.NT_INTERNAL
        self.m_fDistanceToSurface = 0.0

class RFVoxelOctree:
    def __init__(self, origin, voxel_size, dim):
        self.m_pkRoot = RFVoxelOctreeNode()
        self.m_kDataDimension = RFVector3i(dim, dim, dim)
        self.m_kOctreeOrigin = RFVector3d(*origin)
        self.m_fVoxelSize = voxel_size
        self.m_uiOctreeDim = dim
        self.m_uiOctreeDepth = int(log2(dim))
        self.m_uiNumberOfNodes = 1
        self.m_uiNumberOfLeafs = 0
        self.m_uiNumberOfEmptyLeafs = 0
        self.m_uiNumberOfInternalNodes = 0

    def insert_point(self, point):
        # Not given in the sources.
        pass

# CGAL::Orthtree Usage [S2]
# Defined as:
# CGAL::Orthtree< Orthtree_traits_point< GeomTraits, PointRange, PointMap, cubic_nodes, 3 > > Octree;
# Traversal:
# for leaf in Octree.Leaves_traversal():
#     # Process leaf
#     pass
# Output:
# Octree.dump_to_polylines(output_stream)
```

### Parameters
*   `m_fVoxelSize`: Physical size of a voxel. Default: 1.0f [S1, unofficial].
*   `m_uiOctreeDim`: Total dimension of the voxel grid. Depth = $\log_2(m\_uiOctreeDim)$ [S1, unofficial].
*   `isovalue`: Threshold for isosurfacing (e.g., 0.3 in CGAL examples) [S2].

### Edge Cases and Pitfalls
*   **Cracks:** If nodes are split with uniform size around the surface but varying depths exist around the isosurface, cracks (boundaries) will appear in the resulting mesh [S2].
*   **Refinement Strategy:** Dense refinement around the isosurface ensures a manifold surface, whereas arbitrary refinement can lead to non-manifold surfaces [S2].
*   **Precision:** CGAL examples use `oo.precision(17)` for output polylines [S2].

## Subtopics

### [Octree Construction for Point Clouds](1070-octree-construction-for-point-clouds.md)
The parent method describing the hierarchical subdivision process.

### [Octree-based Level of Detail and Culling](1079-octree-based-level-of-detail-and-culling.md)
A sibling method using the octree for rendering optimization rather than spatial indexing for surface extraction.

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, unofficial][S2].
*   **Memory Complexity:** Not given in the sources [S1, unofficial][S2].
*   **Strengths:** Integrates with CGAL isosurfacing algorithms (MC, TMC, DC) [S2].
*   **Limitations:** Varying depths can cause mesh cracks if not handled by adaptive refinement strategies [S2].
*   **When to use:** When organizing point clouds for surface extraction or spatial queries requiring hierarchical access [S1, unofficial][S2].
*   **When not to use:** Not given in the sources [S1, unofficial][S2].

## Background
The CGAL::Orthtree implementation originated from the 2022 Google Summer of Code, contributed by Julian Stahl, mentored by Daniel Zint and Pierre Alliez [S2]. It provides first implementations of Marching Cubes, Topologically Correct Marching Cubes, and Dual Contouring using octrees [S2].

## Sources
[S1] itsyouyi/Voxelizer, https://github.com/itsyouyi/Voxelizer, fetched 2026-10-27 (unofficial)
[S2] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
