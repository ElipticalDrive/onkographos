# Voxel Occupancy Test

Also called: Inclusion Test, Void Check, Voxel Occlusion Query

## Overview
The Voxel Occupancy Test determines whether a given continuous 3D point lies within an occupied voxel of a discrete voxel grid. It solves the problem of spatial querying for point-cloud or mesh data represented as a voxel structure, returning a binary result for containment. It differs from [Point-in-Box Inclusion Test](1400-point-in-box-inclusion-test.md) by operating on a pre-constructed sparse or dense voxel grid structure rather than a simple axis-aligned bounding box, and from [Voxel Filled Check](1705-voxel-filled-check.md) by accepting arbitrary continuous coordinates rather than discrete voxel indices.

## How it works
**Inputs:**
1.  A voxel grid data structure (e.g., `voxel_world` texture or sparse grid).
2.  A set of query points, typically represented as an $(N, 3)$ array of floating-point coordinates `pos` [S1].

**Outputs:**
1.  A boolean array of length $N$, where each element indicates if the corresponding point is inside an occupied voxel [S1].

**Data Structures:**
*   **Voxel Grid:** A 3D texture or array where each voxel stores occupancy data. In the implementation from [S2, unofficial], the voxel data is stored in a texture format where the red channel (`voxel.r`) contains an 8-bit unsigned integer mask (`u8`) representing occupancy [S2, unofficial].
*   **Texture Size:** A 3D vector `voxel_uniforms.texture_size` defining the dimensions of the voxel grid in voxels [S2, unofficial].

**Steps:**
1.  **Coordinate Validation:** For each query point `pos = (x, y, z)`, check if any component is outside the valid grid range. The valid range is defined as $[0, \text{texture\_size})$. Specifically, if $x < 0$, $y < 0$, $z < 0$, $x \ge \text{texture\_size}.x$, $y \ge \text{texture\_size}.y$, or $z \ge \text{texture\_size}.z$, the point is considered outside [S2, unofficial].
2.  **Index Mapping:** If the point is within bounds, map the continuous coordinates to integer voxel indices. The mapping depends on the coordinate system convention. In the source [S2, unofficial], the mapping uses `vec3<i32>(pos.zyx)`, indicating a ZYX ordering for the texture lookup (Z is the first index, Y the second, X the third) [S2, unofficial].
3.  **Voxel Retrieval:** Sample the voxel value at the calculated integer indices from the `voxel_world` texture [S2, unofficial].
4.  **Occupancy Extraction:** Extract the occupancy bit from the retrieved voxel data. In the source [S2, unofficial], this is done by masking the red channel with `0xFFu` (keeping the lower 8 bits) and converting the result to a float [S2, unofficial].
5.  **Normalization:** Clamp the extracted value to the range $[0, 1]$. The formula is $\min(\text{float}(\text{voxel}.r \ \& \ 0xFFu), 1.0)$ [S2, unofficial].
6.  **Result:** Return the normalized value as the boolean result (non-zero is true, zero is false) [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_occupancy_test(voxel_grid, texture_size, query_points):
    """
    Tests if points are within an occupied voxel.
    
    Args:
        voxel_grid: A 3D array or texture storing voxel data.
        texture_size: A tuple (size_x, size_y, size_z) defining the grid dimensions.
        query_points: An (N, 3) array of float coordinates (x, y, z).
        
    Returns:
        A boolean array of length N.
    """
    results = []
    
    for pos in query_points:
        x, y, z = pos
        
        # Step 1: Check bounds
        # Valid range is [0, texture_size) for each axis
        if (x < 0.0 or x >= texture_size[0] or
            y < 0.0 or y >= texture_size[1] or
            z < 0.0 or z >= texture_size[2]):
            results.append(False)
            continue
            
        # Step 2: Map to integer indices
        # Note: Source [S2, unofficial] uses ZYX ordering for texture lookup
        idx_z = int(z)
        idx_y = int(y)
        idx_x = int(x)
        
        # Step 3: Load voxel
        # Accessing the voxel at (idx_x, idx_y, idx_z)
        # Assuming voxel_grid is indexed as [x, y, z] or similar, 
        # but the source uses vec3<i32>(pos.zyx) which implies 
        # the texture coordinate order is Z, Y, X.
        # We assume a standard 3D array access for clarity, 
        # but the semantic is fetching the voxel at these indices.
        voxel = voxel_grid[idx_x, idx_y, idx_z]
        
        # Step 4 & 5: Extract and normalize occupancy
        # Source [S2, unofficial]: min(f32(voxel.r & 0xFFu), 1.0)
        # Assuming voxel is a struct or tuple with a 'r' component
        # representing the first channel.
        occupancy_raw = voxel.r & 0xFF
        
        # Convert to float and clamp
        occupancy = min(float(occupancy_raw), 1.0)
        
        # Step 6: Result
        results.append(occupancy > 0.0)
        
    return results
```

**Parameters:**
*   `texture_size`: The dimensions of the voxel grid. Typical values depend on the scene resolution [S2, unofficial].
*   `voxel.r`: The red channel of the voxel data, interpreted as an 8-bit integer mask [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** Points exactly on the boundary $x = \text{texture\_size}.x$ are considered outside (exclusive upper bound) [S2, unofficial].
*   **Coordinate Order:** The mapping from continuous $(x,y,z)$ to grid indices may require reordering (e.g., ZYX) depending on the underlying texture or array layout [S2, unofficial].
*   **Data Type Conversion:** Converting integer masks to floats must be done carefully to avoid truncation or sign errors [S2, unofficial].

## Subtopics

### [Point-in-Box Inclusion Test](1400-point-in-box-inclusion-test.md)
Partitions points by orthogonal bounding boxes; used for direct geometric containment checks.

### [3D Filled Circle Kernel Generation](1520-3d-filled-circle-kernel-generation.md)
Generates binary occupancy masks for spheres; used for ray casting optimization.

### [Voxel Filled Check](1705-voxel-filled-check.md)
Checks if a specific discrete voxel is occupied; used for pre-computed voxel masks.

### [Sparse AABB Collision Test](3047-sparse-aabb-collision-test.md)
Tests if an AABB intersects occupied voxels; used for static collision detection.

### [Solidity Check](4229-solidity-check.md)
Determines if a coordinate is solid or void; used for final occupancy resolution with boundary handling.

### [Sphere-Clamped Voxel Neighbor Search](5131-sphere-clamped-voxel-neighbor-search.md)
Searches for neighbors within a sphere; used for local neighborhood queries.

### [Voxel Collision Test](5774-voxel-collision-test.md)
Tests if a solid voxel exists in a region; used for physics obstruction detection.

### [Mask Cache Lookup](6434-mask-cache-lookup.md)
Looks up occupancy in a boolean grid; used for efficient coordinate transformation and bounds checking.

### [Voxel Visibility Culling](8235-voxel-visibility-culling.md)
Discards occluded voxels; used for rendering optimization.

## Operating characteristics
*   **Time Complexity:** $O(N)$ for $N$ query points, as each point requires a constant number of arithmetic operations and a single texture/array lookup [S1][S2, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary space per query, assuming the voxel grid is pre-loaded [S1][S2, unofficial].
*   **Performance:** Performance is dominated by memory access patterns to the voxel grid. Cache locality depends on the spatial coherence of query points [S2, unofficial].
*   **Strengths:** Simple to implement; efficient for sparse queries on dense grids if the grid is small enough to fit in cache [S1][S2, unofficial].
*   **Limitations:** Does not support acceleration structures for large sparse grids; performance degrades if the voxel grid is too large for efficient memory access [S2, unofficial].
*   **When to use:** When querying a moderate number of points against a voxel grid that fits in memory [S1][S2, unofficial].
*   **When not to use:** For very large sparse grids where acceleration structures (like BVHs or sparse voxel octrees) are needed [S2, unofficial].

## Background
The Voxel Occupancy Test is a fundamental operation in voxel-based geometry processing. It is commonly used in conjunction with voxelization techniques to determine point containment within a discretized volume [S1]. The method is implemented in libraries such as Open3D, where it is exposed as `check_if_included` [S1]. The specific implementation details regarding texture sampling and bit masking are found in engine-level implementations like the Bevy Voxel Engine [S2, unofficial].

## Sources
[S1] Voxelization — Open3D 0.12.0 documentation, https://www.open3d.org/docs/0.12.0/tutorial/geometry/voxelization.html, fetched 2026-10-26
[S2] ria8651/bevy-voxel-engine, https://github.com/ria8651/bevy-voxel-engine, fetched 2026-10-27 (unofficial)
