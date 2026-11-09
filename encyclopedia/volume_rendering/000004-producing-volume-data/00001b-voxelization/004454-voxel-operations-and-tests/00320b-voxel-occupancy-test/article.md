# Voxel Occupancy Test

Also called: Inclusion Test, Void Check, Voxel Occlusion Query

## Overview
The Voxel Occupancy Test determines whether query points lie within occupied voxels of a pre-computed voxel grid. It solves the problem of fast spatial containment queries for point clouds or particle systems against a discrete volumetric representation. Unlike general point-in-box tests that check continuous bounding volumes, this method operates on a discrete grid structure, often optimized via in-place partitioning or direct texture lookups to support subsequent sorting or hierarchical decomposition.

## How it works
**Inputs and Outputs:**
- Input: An array of $N$ query points, typically represented as an $(N, 3)$ array of floating-point coordinates $P = \{p_0, p_1, \dots, p_{N-1}\}$ where $p_i = (x_i, y_i, z_i)$.
- Input: A voxel grid data structure (e.g., a sparse hash map or a dense texture buffer) representing occupied space.
- Input: The voxel grid dimensions, specifically the texture size $S = (S_x, S_y, S_z)$, defining the bounds $[0, S_x) \times [0, S_y) \times [0, S_z)$.
- Output: A boolean array $B$ of length $N$, where $B[i]$ is `true` if $p_i$ is within an occupied voxel, and `false` otherwise.

**Data Structures:**
The voxel grid is typically stored as a 3D texture or a structured array. In implementations using texture buffers (e.g., GPU-based or Open3D-style), the voxel data is accessed via integer coordinates. The texture size $S$ defines the valid index range. Each voxel may store a bitmask or a scalar value indicating occupancy.

**Steps:**
1. **Coordinate Transformation:** For each query point $p_i = (x_i, y_i, z_i)$, determine the integer voxel indices $(u, v, w)$ corresponding to the voxel containing the point. This usually involves scaling the world coordinates by the voxel size and truncating to integers. If the voxel grid is axis-aligned and covers the world origin, this is:
   $$ u = \lfloor x_i / \text{voxel\_size} \rfloor, \quad v = \lfloor y_i / \text{voxel\_size} \rfloor, \quad w = \lfloor z_i / \text{voxel\_size} \rfloor $$
   [G] If the grid has an offset, add the offset before truncation.

2. **Bounds Check:** Verify that the computed indices $(u, v, w)$ lie within the valid range of the voxel grid. If any component is less than 0 or greater than or equal to the corresponding dimension of the texture size $S$, the point is outside the grid.
   $$ \text{if } u < 0 \lor u \ge S_x \lor v < 0 \lor v \ge S_y \lor w < 0 \lor w \ge S_z \implies B[i] = \text{false} $$

3. **Voxel Lookup:** If the point is within bounds, access the voxel data at the integer coordinates. In many implementations (e.g., Open3D, Bevy Voxel Engine), the coordinates are accessed in a specific order (e.g., $z, y, x$ for memory layout or texture sampling).
   $$ \text{voxel\_data} = \text{voxel\_world}[\text{vec3<i32>}(w, v, u)] $$
   [S2, unofficial] Note: Some implementations use `vec3<i32>(pos.zyx)` to map world coordinates to texture indices.

4. **Occupancy Evaluation:** Determine if the voxel is occupied. This may involve checking a specific bit, comparing a scalar value, or masking the result.
   $$ \text{is\_occupied} = \min(\text{f32}(\text{voxel\_data}.r \ \& \ 0xFFu), 1.0) $$
   [S2, unofficial] If `is_occupied` is non-zero, $B[i] = \text{true}$; otherwise, $B[i] = \text{false}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_occupancy_test(queries, voxel_grid, texture_size, voxel_size=1.0):
    """
    queries: list of (x, y, z) tuples
    voxel_grid: 3D array or texture buffer
    texture_size: tuple (Sx, Sy, Sz)
    voxel_size: float, size of each voxel in world units
    """
    results = []
    for p in queries:
        x, y, z = p
        
        # Step 1: Coordinate Transformation
        # Assuming grid starts at 0,0,0. If offset exists, adjust here.
        u = int(x / voxel_size)
        v = int(y / voxel_size)
        w = int(z / voxel_size)
        
        # Step 2: Bounds Check
        if u < 0 or u >= texture_size[0] or \
           v < 0 or v >= texture_size[1] or \
           w < 0 or w >= texture_size[2]:
            results.append(False)
            continue
            
        # Step 3: Voxel Lookup
        # Accessing voxel data. Note: Some APIs use zyx order for texture sampling.
        # Here we assume a standard 3D array access [x][y][z] or [z][y][x] depending on layout.
        # For Open3D-style, the internal structure handles the mapping.
        # For a generic implementation, we assume direct access.
        try:
            voxel_data = voxel_grid[u][v][w]
        except IndexError:
            results.append(False)
            continue
            
        # Step 4: Occupancy Evaluation
        # Check if the voxel is occupied. 
        # Example: checking if the first byte is non-zero or if a specific bit is set.
        # Using the formula from [S2, unofficial]: min(f32(voxel.r & 0xFFu), 1.0)
        # Assuming voxel_data is an integer or a struct with a 'r' field.
        if isinstance(voxel_data, int):
            is_occupied = min(voxel_data & 0xFF, 1) != 0
        elif hasattr(voxel_data, 'r'):
            is_occupied = min(voxel_data.r & 0xFF, 1) != 0
        else:
            # Generic fallback: non-zero value implies occupancy
            is_occupied = bool(voxel_data)
            
        results.append(is_occupied)
        
    return results
```

**Parameters:**
- `voxel_size`: The physical size of a single voxel. Typical values depend on the scene scale (e.g., 0.05 for high-res scans, 1.0 for coarse grids).
- `texture_size`: The dimensions of the voxel grid. Must match the resolution used during voxelization.

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point division and truncation can lead to off-by-one errors at boundaries. Use consistent rounding (e.g., floor) for both voxelization and query.
- **Boundary Conditions:** Points exactly on the boundary ($x = S_x \cdot \text{voxel\_size}$) should be handled consistently (usually excluded, as indices are $< S_x$).
- **Empty Voxels:** If the voxel grid is sparse, ensure the lookup mechanism handles missing voxels gracefully (returning `false` or throwing an error that is caught).

## Subtopics

### [Point-in-Box Inclusion Test](1400-point-in-box-inclusion-test.md)
The Point-in-Box Inclusion Test partitions a set of points by determining which lie within an orthogonal bounding box. It serves as the core primitive for voxel occupancy tests, returning the count of points inside or on the boundary of the box. Unlike sibling methods that may use spatial acceleration structures or kernel approximations, this method performs a direct geometric containment check, often optimized via in-place partitioning to support subsequent sorting or hierarchical decomposition.

### [3D Filled Circle Kernel Generation](1520-3d-filled-circle-kernel-generation.md)
This method generates a 3D binary kernel where elements within a specified radius from the center are set to 1, and others are set to 0, with the center element explicitly set to 0. It solves the problem of defining a discrete spherical occupancy region for voxel-based operations [G]. Not given in the sources are comparisons to sibling methods.

### [Voxel Filled Check](1705-voxel-filled-check.md)
Voxel Filled Check determines whether a specific voxel, brick, or chunk within a structured volume grid is occupied by geometry or data. It acts as a binary occupancy test for pre-computed voxel masks.

### [Sparse AABB Collision Test](3047-sparse-aabb-collision-test.md)
The Sparse AABB Collision Test determines whether an Axis-Aligned Bounding Box (AABB) intersects any occupied voxels within a sparse volume representation. It solves static collision detection for volumetric objects by iterating through the voxel grid defined by the AABB's bounds. It differs from its sibling [Point-in-Box Inclusion Test](1400-point-in-box-inclusion-test.md) by evaluating a continuous volume rather than discrete points, and from [Voxel Filled Check](1705-voxel-filled-check.md) by supporting heterogeneous voxel types (solid, liquid, unknown) and caching terrain noise evaluation per column to optimize performance [G].

### [Solidity Check](4229-solidity-check.md)
Solidity Check determines whether a specific voxel coordinate in a 3D world is occupied by solid geometry or is empty space (void). It serves as the final step in voxel occupancy testing, resolving whether a point lies within a solid volume. It differs from general occupancy tests by explicitly handling boundary conditions, coordinate wrapping, and chunk-based data structures to prevent rendering artifacts at world edges.

### [Sphere-Clamped Voxel Neighbor Search](5131-sphere-clamped-voxel-neighbor-search.md)
Sphere-Clamped Voxel Neighbor Search finds the closest occupied voxel cell to a query point within a spherical radius, constrained to the grid boundaries. It differs from [Point-in-Box Inclusion Test](1400-point-in-box-inclusion-test.md) by returning the nearest neighbor rather than a binary occupancy result, and from [3D Filled Circle Kernel Generation](1520-3d-filled-circle-kernel-generation.md) by performing a runtime search for a specific query point rather than precomputing a static kernel. It serves as the spatial lookup primitive for [PBF Displacement Calculation with Grid-Based Neighbor Search](9026-pbf-displacement-calculation-with-grid-based-neighbor-search.md).

### [Voxel Collision Test](5774-voxel-collision-test.md)
A method to determine if a vertical column of voxels at a specific world XZ position contains any solid geometry between two world Y heights ($slab_{lo}$ and $slab_{hi}$). It solves the problem of collision detection in voxel-based environments, specifically allowing entities to walk through carved-out holes or gaps in terrain. Unlike general point-in-voxel tests which check a single coordinate, this method iterates along the Y-axis to detect occupancy within a vertical slab.

### [Mask Cache Lookup](6434-mask-cache-lookup.md)
Mask Cache Lookup looks up occupancy information in a 3D boolean grid for a set of 3D points, converting world coordinates to grid indices. It differs from [Point-in-Box Inclusion Test](1400-point-in-box-inclusion-test.md) by using a discrete grid structure rather than continuous bounding volumes, and from [Voxel Filled Check](1705-voxel-filled-check.md) by explicitly handling the coordinate transformation and bounds checking in a single pass [G].

### [Voxel Visibility Culling](8235-voxel-visibility-culling.md)
Voxel Visibility Culling is a post-processing step in voxel-based rendering that discards voxels which do not contribute to the final image, thereby reducing the instance count and rendering load. It operates by evaluating the scalar field value at the voxel center relative to a target isovalue to determine if the voxel lies on the visible surface. It is distinct from [Voxel Occupancy Test](1281-voxel-occupancy-test.md), which only determines if a point is within the occupied volume, by specifically filtering for surface-adjacent or surface-contained voxels based on scalar thresholds.

## Operating characteristics
- **Time Complexity:** $O(N)$ for $N$ query points, assuming $O(1)$ voxel lookup. If the grid is sparse and uses a hash map, the average case is $O(1)$ but worst-case can be $O(K)$ due to collisions.
- **Memory Complexity:** Depends on the voxel grid storage. Dense grids use $O(S_x S_y S_z)$ memory. Sparse grids use $O(V)$ where $V$ is the number of occupied voxels.
- **Measured Performance:** In Open3D, the `check_if_included` method is optimized for bulk queries on CPU, leveraging vectorized operations. On GPU, texture lookups are extremely fast but require data transfer overhead.
- **Strengths:** Fast, simple, and robust for static scenes. No complex data structure traversal required.
- **Limitations:** Only works for points. Does not handle ray queries or continuous geometry intersection. Precision issues at boundaries.
- **When to Use:** When you have a set of discrete points and need to check their occupancy against a voxelized model. Ideal for particle systems, point cloud filtering, or collision detection for discrete entities.
- **When Not to Use:** For ray tracing, continuous geometry intersection, or when the query points are not aligned with the voxel grid.

## Background
The Voxel Occupancy Test is a fundamental operation in voxel-based graphics and simulation. It originated with the development of voxelization techniques for rendering and collision detection. Open3D popularized this method in its geometry module, providing a high-level API for point cloud processing. The Bevy Voxel Engine also implements a similar test for game physics, demonstrating its utility in real-time applications.

## Sources
[S1] Voxelization — Open3D 0.12.0 documentation, https://www.open3d.org/docs/0.12.0/tutorial/geometry/voxelization.html, fetched 2026-10-26
[S2] ria8651/bevy-voxel-engine, https://github.com/ria8651/bevy-voxel-engine, fetched 2026-10-27 (unofficial)
