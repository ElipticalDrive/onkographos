# Voxel-Cell Duality Mapping

Also called: Cell Primitive Definition, Logical Shift Mapping, Logical Shift for Ray Tracing, Scalar Field Remapping

## Overview
Voxel-Cell Duality Mapping is a technique for ray tracing isosurfaces defined within cells of 8 voxels using an octree that stores scalar values at voxel centers [S1, S2][S3, unofficial]. It establishes a dual relationship between voxels and cells by logically shifting scalar positions backward by half a voxel width, allowing the ray tracer to intersect 8-voxel cell primitives defined by the stored leaf values [S1, S2][S3, unofficial]. Unlike methods that expand storage to hold complete cell data, this method retrieves forward neighbors at intersection time, maintaining single scalar storage per leaf [S2][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An octree structure storing scalar values at voxel centers, where each leaf node contains a single scalar datum $v$ [S1, S2][S3, unofficial].
*   **Output:** Intersection points with an isosurface defined within cells formed by 8 adjacent voxels [S1, S2][S3, unofficial].

**Data Structures:**
*   **Octree:** Nodes contain scalar values at their centers, and leaf nodes store a single scalar value $v$ [S1, S2][S3, unofficial].
*   **Cell Primitive:** A box defined by 8 vertices, where the vertices correspond to the mapped positions of 8 adjacent voxels in the octree [S1, S2][S3, unofficial].

**Mapping Logic:**
1.  **Logical Shift:** The position of all scalars is logically shifted backward by half a unit of voxel width to re-map the scalar field from voxel centers to cell corners [S1, S2][S3, unofficial].
2.  **Corner Mapping:** Each voxel is mapped to the lower-left corner (minimum vertex) of a cell [S1, S2][S3, unofficial].
3.  **Boundary Condition:** Values outside the octree data boundary are defined to be zero [S1, S2][S3, unofficial].

**Intersection Process:**
When a ray traverses an octree leaf node at indices $(i, j, k)$ with a datum $v$, the ray intersects a box (cell) where $(i, j, k)$ is mapped to the minimum vertex, and the remaining 7 vertices are formed by the forward neighbors of the voxel at $(i, j, k)$ in the octree [S1][S3, unofficial].

**Neighbor Retrieval:**
Instead of expanding each voxel to contain its forward neighbors (which would require 8 times the storage), the method retrieves the 7 forward neighbors of a voxel at intersection time [S1, S2][S3, unofficial].
*   For a volume stored in a 3D array, querying these neighbors is a trivial array index operation [S1, S2][S3, unofficial].
*   For an octree, this requires point location to retrieve the voxel values of the forward neighbors, where full top-down point location from the root is $O(\log(N))$ but neighbor-finding techniques can significantly reduce this cost [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def intersect_cell(ray_origin, ray_direction, octree, isovalue):
    """
    Intersect a ray with an isosurface defined by voxel-cell duality.
    """
    # Standard octree traversal to find the leaf node intersected by the ray
    leaf_node = octree.traverse(ray_origin, ray_direction)
    
    if leaf_node is None:
        return None
    
    # Get the scalar value at the current leaf node (voxel center)
    v_center = leaf_node.scalar_value
    
    # Determine the indices (i, j, k) of the current leaf node
    i, j, k = leaf_node.indices
    
    # Helper function to get scalar value, returning 0 for out-of-bounds
    def get_scalar(idx_i, idx_j, idx_k):
        # Standard octree point location to find the node at (idx_i, idx_j, idx_k)
        node = octree.point_location(idx_i, idx_j, idx_k)
        if node is None:
            return 0.0  # Boundary condition: values outside octree are zero
        return node.scalar_value
    
    # Retrieve the 8 corner values for the cell
    # The current voxel is mapped to the minimum vertex (lower-left corner)
    # Neighbors are at (i+1, j, k), (i, j+1, k), (i, j, k+1), etc.
    corners = [
        get_scalar(i,   j,   k),     # Current voxel (min corner)
        get_scalar(i+1, j,   k),     # Neighbor in x
        get_scalar(i,   j+1, k),     # Neighbor in y
        get_scalar(i+1, j+1, k),     # Neighbor in x,y
        get_scalar(i,   j,   k+1),   # Neighbor in z
        get_scalar(i+1, j,   k+1),   # Neighbor in x,z
        get_scalar(i,   j+1, k+1),   # Neighbor in y,z
        get_scalar(i+1, j+1, k+1)    # Neighbor in x,y,z (max corner)
    ]
    
    # Standard trilinear ray-cube intersection or Marching Cubes is applied 
    # to the 8 corner values to find the isosurface intersection.
    # The sources do not specify the exact reconstruction algorithm.
```

**Parameters:**
*   **Isovalue:** The threshold value for the isosurface extraction [G]. Not given in the sources.
*   **Voxel Width:** The unit of voxel width used for the logical shift [G]. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Storage Expansion:** Storing each cell completely by expanding each voxel to contain its forward neighbors requires 8 times the storage of the original volume, which contradicts the goal of ray tracing compressed data [S1, S2][S3, unofficial].
*   **Boundary Handling:** Values outside the octree data boundary are defined to be zero [S1, S2][S3, unofficial].
*   **Performance:** For octrees, retrieving forward neighbors requires point location, which is more intensive than array indexing [S1, S2][S3, unofficial]. Full top-down point location is $O(\log(N))$, but neighbor-finding techniques can reduce this cost [S3, unofficial].

## Subtopics

### [Spherical Volume Ray Walking](3865-spherical-volume-ray-walking.md)
Ray walking in spherical coordinates. Not given in the sources.

### [Regular Grid](3950-regular-grid.md)
Ray tracing on a regular grid structure. Not given in the sources.

### [Plane Marching](3959-plane-marching.md)
Marching planes for volume rendering. Not given in the sources.

### [Voxel-based Ray Tracing for GI and Specular Reflections](4091-voxel-based-ray-tracing-for-gi-and-specular-reflections.md)
Voxel-based ray tracing for global illumination and specular reflections. Not given in the sources.

### [Optimised Voxel Ray Marching](4240-optimised-voxel-ray-marching.md)
Optimized voxel ray marching techniques. Not given in the sources.

## Operating characteristics
*   **Time Complexity:** Neighbor retrieval in an octree is more intensive than in a 3D array, with full top-down point location at $O(\log(N))$ reduced by neighbor-finding techniques [S3, unofficial].
*   **Memory Complexity:** Single scalar storage per leaf node, avoiding the 8x storage expansion required by naive cell expansion [S1, S2][S3, unofficial].
*   **Strengths:** Allows ray tracing of compressed octree data without expanding storage, and permits traversal of interior nodes of the octree volume [S1, S2][S3, unofficial].
*   **Limitations:** Neighbor retrieval in octrees is more complex and potentially slower than in regular grids [S1, S2][S3, unofficial].
*   **When to Use:** When ray tracing isosurfaces from compressed octree volumes where storage efficiency is critical [S1, S2][S3, unofficial].
*   **When Not to Use:** When neighbor retrieval performance is critical and the data is stored in a regular grid, where array indexing is trivial [S1, S2][S3, unofficial].

## Background
Voxel-Cell Duality Mapping was introduced in the context of interactive isosurface ray tracing of large octree volumes [S1, S2][S3, unofficial]. The method leverages the dual relationship between voxels and cells to efficiently ray trace isosurfaces defined within cells of 8 voxels [S1, S2][S3, unofficial]. The approach of retrieving forward neighbors at intersection time is attributed to Parker et al. [S1, S2].

## Sources
[S1] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~wald/Publications/2006/OctIso/octiso.pdf, fetched 2026-10-27
[S2] PDF Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~knolla/octiso-rt06.pdf, fetched 2026-10-28
[S3] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
