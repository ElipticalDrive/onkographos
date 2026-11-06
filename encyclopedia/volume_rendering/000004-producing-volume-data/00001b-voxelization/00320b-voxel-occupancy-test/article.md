# Voxel Occupancy Test

Also called: Inclusion Test, Void Check, Voxel Occlusion Query

## Overview
The Voxel Occupancy Test determines whether a given set of continuous world-space points lies within any occupied voxel of a pre-computed voxel grid [S1]. It solves the problem of spatial querying for point clouds or particle systems against a discrete volumetric representation, returning a boolean mask indicating inclusion [S1]. Unlike [Rasterization-based Voxelization](908-rasterization-based-voxelization.md) which projects geometry to determine occupancy, this method performs a direct lookup against an existing grid structure, serving as the inverse operation of voxel generation [S1].

## How it works
**Inputs and Outputs:**
- Input: An array of $N$ query points, represented as an $(N, 3)$ array of floating-point coordinates $P = \{p_0, p_1, \dots, p_{N-1}\}$, where each $p_i = (x_i, y_i, z_i)$ is in world space [S1].
- Input: A voxel grid data structure containing occupancy information and grid parameters (origin, size, resolution) [S1].
- Output: A boolean array $B$ of length $N$, where $B[i]$ is `true` if point $p_i$ is contained within an occupied voxel, and `false` otherwise [S1].

**Data Structures:**
The voxel grid is typically stored as a sparse or dense structure mapping integer grid indices $(i, j, k)$ to occupancy status. In the Open3D implementation, the grid maintains a list of occupied voxel coordinates [S1]. In GPU-based implementations (e.g., Bevy Voxel Engine), occupancy is often stored in a texture or buffer where each voxel is represented by a byte or bit [S2, unofficial].

**Steps:**
1.  **Coordinate Mapping**: For each query point $p_i = (x_i, y_i, z_i)$, map the continuous world coordinates to discrete integer grid indices $(idx_x, idx_y, idx_z)$. This requires the grid's origin $O = (o_x, o_y, o_z)$ and voxel size $s$ [S1, S3, unofficial]:
    $$ idx_x = \lfloor \frac{x_i - o_x}{s} \rfloor $$
    $$ idx_y = \lfloor \frac{y_i - o_y}{s} \rfloor $$
    $$ idx_z = \lfloor \frac{z_i - o_z}{s} \rfloor $$
    [G] Note: The specific mapping formula depends on the grid's coordinate system. Open3D uses a standard floor mapping relative to the grid's bounds [S1].

2.  **Bounds Checking**: Verify that the computed grid indices are within the valid range of the voxel grid. If the grid has dimensions $W \times H \times D$, the indices must satisfy:
    $$ 0 \le idx_x < W, \quad 0 \le idx_y < H, \quad 0 \le idx_z < D $$
    If any index is out of bounds, the point is not included [S2, unofficial].

3.  **Occupancy Lookup**: Check if the voxel at $(idx_x, idx_y, idx_z)$ is marked as occupied.
    - In a sparse list-based grid (Open3D), check if $(idx_x, idx_y, idx_z)$ exists in the set of occupied voxels [S1].
    - In a texture-based grid (Bevy), load the voxel data at the integer coordinates. The source code indicates a specific memory layout where the voxel is accessed via `vec3<i32>(pos.zyx)` (note the ZYX swizzle, implying a specific memory layout or coordinate convention) [S2, unofficial]. The occupancy is extracted from the red channel (`voxel.r`) masked with `0xFFu` and clamped to 1.0 [S2, unofficial].

4.  **Result Assignment**: Set $B[i]$ to `true` if the voxel is occupied, `false` otherwise.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_occupancy_test(points, voxel_grid):
    """
    points: list of (x, y, z) tuples or (N, 3) array
    voxel_grid: object containing occupied_voxels (set of (i,j,k)) and bounds
    returns: list of booleans
    """
    results = []
    # Extract grid parameters from voxel_grid
    # Note: Open3D's check_if_included handles the internal mapping [S1]
    # This pseudocode assumes a generic sparse grid structure
    
    occupied_set = voxel_grid.occupied_voxels
    bounds = voxel_grid.bounds # (min_x, min_y, min_z, max_x, max_y, max_z)
    voxel_size = voxel_grid.voxel_size
    
    for p in points:
        x, y, z = p
        
        # 1. Map to grid indices
        # Assuming origin is at bounds.min and standard floor mapping
        idx_x = int((x - bounds.min_x) // voxel_size)
        idx_y = int((y - bounds.min_y) // voxel_size)
        idx_z = int((z - bounds.min_z) // voxel_size)
        
        # 2. Bounds check
        if not (0 <= idx_x < bounds.width and 
                0 <= idx_y < bounds.height and 
                0 <= idx_z < bounds.depth):
            results.append(False)
            continue
            
        # 3. Occupancy check
        if (idx_x, idx_y, idx_z) in occupied_set:
            results.append(True)
        else:
            results.append(False)
            
    return results
```

**Parameters:**
- `voxel_size`: The physical size of a single voxel. Must match the resolution used during voxelization [S1].
- `origin`: The world-space coordinate of the grid's minimum corner [S1].

**Edge Cases and Pitfalls:**
- **Precision**: Floating-point rounding errors in coordinate mapping can cause points on voxel boundaries to be assigned to incorrect voxels. Using consistent floor/ceil logic is critical [S1, S3, unofficial].
- **Out-of-Bounds**: Points outside the grid bounds must be handled explicitly to avoid memory access errors [S2, unofficial].
- **Coordinate Swizzling**: In GPU implementations, the order of indices (XYZ vs ZYX) must match the memory layout of the voxel texture/buffer [S2, unofficial].

## Subtopics

### [Point-in-Box Inclusion Test](1400-point-in-box-inclusion-test.md)
The Point-in-Box Inclusion Test determines which points from a given array lie within an orthogonal bounding box. It serves as the core primitive for [Voxel Occupancy Test](1281-voxel-occupancy-test.md), filtering points to identify voxel occupancy. Unlike [3D Filled Circle Kernel Generation](1520-3d-filled-circle-kernel-generation.md) which uses distance metrics, this method relies on strict axis-aligned boundary checks. It differs from [Sparse AABB Collision Test](3047-sparse-aabb-collision-test.md) by operating on a static point array rather than dynamic collision shapes.

### [Voxel Filled Check](1705-voxel-filled-check.md)
Voxel Filled Check determines whether a specific voxel, brick, or chunk within a structured volume grid is occupied by geometry or data. It serves as the fundamental occupancy test in voxelized volume rendering, allowing ray tracers to skip empty space and accelerate traversal. It differs from [Point-in-Box Inclusion Test](1400-point-in-box-inclusion-test.md) by operating on discrete integer grid indices and bitmask structures rather than continuous spatial queries, and from [Voxel Occupancy Test](1281-voxel-occupancy-test.md) by specifically focusing on the binary state of pre-computed occupancy masks (bits) rather than general point containment.

### [Sparse AABB Collision Test](3047-sparse-aabb-collision-test.md)
The Sparse AABB Collision Test determines whether an Axis-Aligned Bounding Box (AABB) intersects any occupied voxels within a sparse volume representation. It solves static collision detection for volumetric objects by iterating through the voxel grid defined by the AABB's bounds. It differs from its sibling [Point-in-Box Inclusion Test](1400-point-in-box-inclusion-test.md) by evaluating a continuous volume rather than discrete points, and from [Voxel Filled Check](1705-voxel-filled-check.md) by supporting heterogeneous voxel types (solid, liquid, unknown) and caching terrain noise evaluation per column to optimize performance [G].

### [Solidity Check](4229-solidity-check.md)
Solidity Check determines if a specific voxel coordinate in a 3D world is occupied by solid geometry or is void (empty). It solves the problem of boundary condition handling during voxel traversal, ensuring that out-of-bounds regions are treated consistently to prevent rendering artifacts. It differs from its sibling [Point-in-Box Inclusion Test](1400-point-in-box-inclusion-test.md) by explicitly managing chunk-based spatial indexing and asymmetric boundary wrapping for X/Y versus Z axes, rather than simple AABB containment.

### [Sphere-Clamped Voxel Neighbor Search](5131-sphere-clamped-voxel-neighbor-search.md)
Sphere-Clamped Voxel Neighbor Search finds the closest voxel cell to a query point within a specified radius, constrained to the grid boundaries. It solves the problem of locating the nearest occupied or valid material cell in a voxelized environment for operations like ray casting or neighbor lookups. It differs from [Point-in-Box Inclusion Test](1400-point-in-box-inclusion-test.md) by using a spherical distance metric and from [Voxel Filled Check](1705-voxel-filled-check.md) by returning the specific closest cell rather than a boolean occupancy status.

### [Voxel Collision Test](5774-voxel-collision-test.md)
A method to determine if a vertical column of voxels at a specific world XZ position contains any solid geometry between two world Y heights ($slab_{lo}$ and $slab_{hi}$). It solves the problem of collision detection in voxel-based environments, specifically allowing entities to walk through carved-out holes or gaps in terrain. Unlike general point-in-voxel tests which check a single coordinate, this method iterates along the Y-axis to detect occupancy within a vertical slab.

### [Mask Cache Lookup](6434-mask-cache-lookup.md)
Mask Cache Lookup tests if query points lie within occupied voxels by indexing a 3D boolean occupancy grid. It converts world coordinates to grid indices and performs a direct memory lookup. It differs from [Point-in-Box Inclusion Test](1400-point-in-box-inclusion-test.md) by using a discrete grid structure rather than continuous bounding volumes, and from [Voxel Filled Check](1705-voxel-filled-check.md) by explicitly handling the coordinate transformation and bounds checking in a single pass.

## Operating characteristics
- **Time Complexity**: $O(N)$ for $N$ query points, assuming $O(1)$ average-case lookup in the occupancy data structure (e.g., hash set or direct array access) [S1].
- **Memory Complexity**: Depends on the voxel grid storage. Sparse grids use $O(V)$ where $V$ is the number of occupied voxels [S1]. Dense grids use $O(W \times H \times D)$ [S1].
- **Performance**: The test is extremely fast compared to ray-casting or geometric intersection tests, as it avoids floating-point intersection calculations [S1]. The primary cost is the coordinate transformation and memory lookup [S1].
- **Strengths**: Simple implementation, high performance for point queries, compatible with any voxelization method [S1].
- **Limitations**: Only works for points; cannot test arbitrary primitives (lines, triangles) directly without sampling [S1]. Accuracy is limited by voxel resolution [S1].

## Background
The Voxel Occupancy Test is a fundamental operation in voxel-based graphics and simulation. It is commonly used in conjunction with voxelization methods to query the resulting grid [S1]. The method is implemented in libraries such as Open3D, where it is exposed as `check_if_included` [S1]. In game engines like Bevy, similar tests are used for physics and collision detection, often optimized for GPU execution [S2, unofficial].

## Sources
[S1] Voxelization — Open3D 0.12.0 documentation, https://www.open3d.org/docs/0.12.0/tutorial/geometry/voxelization.html, fetched 2026-10-26
[S2] ria8651/bevy-voxel-engine, https://github.com/ria8651/bevy-voxel-engine, fetched 2026-10-27 (unofficial)
