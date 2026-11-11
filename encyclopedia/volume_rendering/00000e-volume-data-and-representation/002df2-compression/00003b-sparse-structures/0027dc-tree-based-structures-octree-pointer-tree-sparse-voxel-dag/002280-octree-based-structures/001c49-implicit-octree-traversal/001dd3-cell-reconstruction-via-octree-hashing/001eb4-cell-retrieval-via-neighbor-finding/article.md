# Cell Retrieval via Neighbor-Finding

Also called: 8-voxel cell intersection, Cell-based ray tracing, Forward-neighbor retrieval, Neighbor-Finding for Cell Retrieval, Neighbor-Finding on Octree, Octree Hashing, Octree Scalar Field Mapping via Forward Neighbors, voxel to cell mapping

## Overview
This method enables isosurface reconstruction (e.g., Marching Cubes) on sparse octree volumes that store only a single scalar value per leaf voxel, by logically mapping each voxel to the lower-left corner of an 8-voxel cell and retrieving the 7 forward neighbors at intersection time. It solves the storage explosion problem of explicitly storing full cells (which would require 8x the original data size) by performing neighbor-finding on the octree structure to assemble the required 8 corner values on the fly. It differs from [Cell Reconstruction via Octree Hashing](1595-cell-reconstruction-via-octree-hashing.md) in its specific application to the "forward-neighbor" retrieval pattern for cell-based primitives, and from [Octree Hashing for Point Location and Neighbor Finding](1984-octree-hashing-for-point-location-and-neighbor-finding.md) by focusing on the runtime assembly of the intersection primitive rather than just the point location mechanism itself.

## How it works

### Inputs and Outputs
*   **Input:** An octree structure where each leaf node stores a single scalar value $v$ at its center. A ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ intersecting the volume.
*   **Output:** A set of 8 scalar values corresponding to the corners of the 8-voxel cell containing the intersection point, used for isosurface reconstruction.

### Data Structures
The octree nodes are standard. Leaf nodes contain a single scalar field. Interior nodes contain pointers to 8 children (octants). The spatial layout assumes a regular grid where voxel width is 1 unit.

### Logical Mapping and Coordinate Shift
The scalar field is defined at voxel centers. To support cell-based intersection (where values are at corners), the field is logically shifted backward by half a voxel width.
*   Let a leaf node be at integer grid coordinates $(i, j, k)$.
*   This voxel is mapped to the minimum vertex (lower-left-front corner) of a cell.
*   The cell spans the range $[i, i+1] \times [j, j+1] \times [k, k+1]$.
*   The 8 corners of this cell correspond to the 8 voxels with indices:
    $$ (i+\delta_x, j+\delta_y, k+\delta_z) $$
    where $\delta_x, \delta_y, \delta_z \in \{0, 1\}$.
*   The target voxel is $(i, j, k)$ ($\delta_x=\delta_y=\delta_z=0$).
*   The 7 forward neighbors are those where at least one $\delta$ is 1.

### Algorithm Steps

1.  **Ray Traversal:** Traverse the octree using standard ray-triangle or ray-box intersection logic for interior nodes. Identify the leaf node $L$ intersected by the ray. Let the grid coordinates of $L$ be $(i, j, k)$.
2.  **Target Value Retrieval:** Retrieve the scalar value $v_{0,0,0}$ stored in leaf $L$.
3.  **Neighbor-Finding:** For each of the 7 forward neighbor directions $(\delta_x, \delta_y, \delta_z) \in \{0,1\}^3 \setminus \{(0,0,0)\}$:
    *   Compute the target grid coordinates $(i+\delta_x, j+\delta_y, k+\delta_z)$.
    *   Perform neighbor-finding on the octree to locate the leaf node at these coordinates.
    *   Retrieve the scalar value $v_{\delta_x, \delta_y, \delta_z}$ from that leaf.
    *   **Edge Case:** If the neighbor coordinates lie outside the bounds of the octree data, define the scalar value as 0 [S4, unofficial].
4.  **Cell Assembly:** Assemble the 8 values $\{v_{\delta_x, \delta_y, \delta_z}\}$ into the cell primitive.
5.  **Intersection:** Perform the isosurface intersection test (e.g., Marching Cubes) on the 8-voxel cell using the retrieved values.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def retrieve_cell_values(octree_root, leaf_coords):
    """
    Retrieves the 8 scalar values for the cell defined by the target leaf.
    leaf_coords: tuple (i, j, k) of the intersected leaf node.
    Returns: list of 8 values [v_000, v_001, v_010, v_011, v_100, v_101, v_110, v_111]
    """
    i, j, k = leaf_coords
    values = []
    
    # Iterate through all 8 combinations of deltas (0 or 1)
    for dx in [0, 1]:
        for dy in [0, 1]:
            for dz in [0, 1]:
                # Compute neighbor coordinates
                ni = i + dx
                nj = j + dy
                nk = k + dz
                
                # Retrieve value from octree
                # If neighbor is outside octree bounds, value is 0
                val = octree_get_value(octree_root, ni, nj, nk)
                values.append(val)
                
    return values

def octree_get_value(root, x, y, z):
    """
    Performs neighbor-finding/point location to retrieve the scalar value
    at grid coordinates (x, y, z).
    Returns 0 if the coordinates are outside the defined octree volume.
    """
    # Implementation depends on the specific neighbor-finding algorithm
    # (e.g., hash-based, pointer-based traversal).
    # This function encapsulates the O(log N) or O(1) lookup.
    leaf = find_leaf_node(root, x, y, z)
    if leaf is None:
        return 0.0
    return leaf.scalar_value
```

### Parameters and Typical Values
*   **Voxel Width:** 1 unit (logical).
*   **Shift:** 0.5 units backward.
*   **Out-of-Bounds Value:** 0.0 [S4, unofficial].

### Pitfalls and Edge Cases
*   **Boundary Conditions:** Neighbors may lie outside the octree. These must be handled by returning a default value (0) rather than crashing [S4, unofficial].
*   **Complexity Skew:** While worst-case neighbor-finding is $O(\log N)$, it skews heavily toward $O(1)$ when neighbors reside in the same parent node [S3]. Implementations should optimize for this common case.
*   **Cache Performance:** Unlike 3D arrays where neighbors are contiguous in memory, octree neighbors are scattered. This can lead to cache misses, making the $O(1)$ skew critical for performance [S3].

## Subtopics

### Octree Hashing for Point Location and Neighbor Finding
*   **What it is:** A specific hashing scheme using binary arithmetic on integer coordinates to accelerate point location and neighbor finding in regular binary trees.
*   **Difference:** Focuses on the underlying hash mechanism for coordinate lookup, whereas Cell Retrieval focuses on the application of that lookup to assemble 8-voxel cells.
*   **When to choose:** When the octree is static and coordinate-based hashing is feasible.
*   **Link:** [Octree Hashing for Point Location and Neighbor Finding](1984-octree-hashing-for-point-location-and-neighbor-finding.md)

### Min/Max Tree Construction with Forward Neighbors
*   **What it is:** A method to compute min/max pairs for interior nodes based on the min/max pairs of their 8 children, specifically for ray tracing cells defined by forward neighbors.
*   **Difference:** This is an acceleration structure construction step that complements cell retrieval, allowing early ray termination.
*   **When to choose:** When ray tracing performance is critical and the volume has significant empty space.
*   **Link:** [Min/Max Tree Construction with Forward Neighbors](2020-min-max-tree-construction-with-forward-neighbors.md)

### Neighbor Cell Access via Hash Table and Morton Code
*   **What it is:** Uses Morton codes and hash tables to map voxel coordinates to values, enabling fast neighbor access.
*   **Difference:** Uses a flat hash table structure rather than a hierarchical octree pointer structure for neighbor lookup.
*   **When to choose:** When memory layout favors flat arrays and Morton code generation is efficient.
*   **Link:** [Neighbor Cell Access via Hash Table and Morton Code](6715-neighbor-cell-access-via-hash-table-and-neighbor-finding.md)

## Operating characteristics

*   **Time Complexity:** Worst-case $O(\log N)$ per neighbor lookup, where $N$ is the number of leaf nodes. In practice, it skews heavily toward $O(1)$ when neighbors are in the same parent node [S3].
*   **Memory Complexity:** No additional storage for neighbor data (unlike the 8x expansion alternative). The octree structure itself is unchanged [S1].
*   **Performance:** Neighbor-finding on octrees is more intensive than array indexing ($O(1)$, cache-friendly) [S3]. It must compete with the performance of uncompressed 3D array lookups.
*   **Strengths:** Allows ray tracing of compressed sparse data without expanding storage. Enables traversal of interior nodes while intersecting well-defined cell primitives [S1].
*   **Limitations:** Higher computational cost per intersection compared to dense grids. Sensitive to cache performance due to scattered memory access.
*   **When to use:** When storage is a constraint and the volume is sparse.
*   **When not to use:** When the volume is dense and fits in memory, or when cache performance is critical and the overhead of neighbor-finding outweighs the benefits of sparsity.

## Background

*   **Origin:** The concept of mapping voxels to cells for isosurface reconstruction in octrees was introduced by Parker et al. [15, 17, 18] in the context of interactive isosurface ray tracing of large octree volumes [S1, S2, S3][S4, unofficial].
*   **History:** Point location and neighbor-finding algorithms were pioneered by Glassner [32] and Samet [105] in ray tracing [S6, unofficial]. Frisken and Perry [26] proposed an efficient hashing scheme using binary arithmetic on integer coordinates, which later works built upon [S6, unofficial].
*   **Usage:** Used in interactive isosurface ray tracing of large volumes, particularly where compression is a goal [S1, S2, S3][S4, unofficial].

## Sources

[S1] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~wald/Publications/2006/OctIso/octiso.pdf, fetched 2026-10-27
[S2] PDF Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~knolla/octiso-rt06.pdf, fetched 2026-10-28
[S3] PDF Interactive Isosurface Ray Tracing of Large Octree Volumes, https://collections.lib.utah.edu/dl_files/1d/58/1d589a03a725c9d894a3866ad32f1386f0613f9d.pdf, fetched 2026-10-28
[S4] Interactive isosurface ray tracing of large octree volumes, https://www.researchgate.net/publication/310054812_Interactive_isosurface_ray_tracing_of_large_octree_volumes, fetched 2026-10-29 (unofficial)
[S5] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.researchgate.net/publication/261082346_Interactive_Isosurface_Ray_Tracing_of_Large_Octree_Volumes, fetched 2026-10-29 (unofficial)
[S6] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
