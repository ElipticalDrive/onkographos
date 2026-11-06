# Sphere Carving

Also called: Carve Sphere, SVO Sphere Carving, SVO modification, Voxel Destruction via SVO, Voxel Sphere Removal, voxel removal

## Overview
Sphere Carving is a voxel occupancy test and modification method that removes solid voxels within a spherical radius from a specified center point in a voxel grid or Sparse Voxel Octree (SVO) [S1, S2, unofficial]. It solves the problem of creating destructible terrain or dynamic object modification by efficiently identifying and clearing voxels intersecting a query sphere [S1, S2, unofficial]. It sets itself apart from its sibling [3D Binary Sphere Kernel Generation](4654-3d-binary-sphere-kernel-generation.md) by operating directly on the storage structure (chunk or SVO) to remove data rather than generating a static kernel, and from [3D Filled Circle Kernel Generation](1520-3d-filled-circle-kernel-generation.md) by supporting arbitrary world-space positions and SVO coordinate transformations [S1, S2, unofficial].

## How it works
Sphere Carving iterates over the axis-aligned bounding box (AABB) of a sphere within a voxel grid, checks each voxel for solidity, and removes those within the radius [S1, S2, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   `center`: The center of the carving sphere. Can be world-space or local-space depending on implementation [S1, S2, unofficial].
    *   `radius`: The radius of the carving sphere in world units [S1, S2, unofficial].
    *   `grid`: The voxel data structure (e.g., a dense chunk array or SVO node) [S1, S2, unofficial].
    *   `vox_size`: The size of a single voxel in world units [S2, unofficial].
    *   `grid_dim`: The dimensions of the grid/chunk [S2, unofficial].
*   **Outputs**:
    *   `removed_count`: The number of solid voxels removed [S1, S2, unofficial].
    *   `gib_positions`: (Optional) A list of world positions of removed voxels, used for spawning debris [S2, unofficial].
    *   `dirty_flag`: A flag indicating the mesh or structure needs rebuilding [S2, unofficial].

### Data Structures
*   **Chunk/Grid**: A 3D array or sparse structure where `grid[x, y, z] != 0` indicates a solid voxel [S1, S2, unofficial].
*   **VoxObject (SVO context)**: Contains `origin` (world center), `rotation` (unused for axis-aligned), `vox_size`, `grid_dim`, `svo` (the octree), `solid_count`, and `dirty` flag [S2, unofficial].

### Steps
1.  **Coordinate Transformation**: If the input `center` is in world space and the grid is local, transform `center` to local grid coordinates [S2, unofficial].
    *   Subtract object origin.
    *   Apply inverse rotation (if non-axis-aligned, though sources note rotation is often unused/axis-aligned) [S2, unofficial].
    *   Convert to voxel indices: $c_x = (x_{local} + \text{extent}/2) / \text{vox\_size}$ [S2, unofficial].
2.  **Radius Conversion**: Convert world radius to grid units: $r_{grid} = \text{radius} / \text{vox\_size}$ [S2, unofficial].
3.  **Bounding Box Calculation**: Determine the integer bounds of the sphere in grid coordinates, clamped to the grid dimensions $[0, \text{grid\_dim}-1]$ [S1, S2, unofficial].
    *   $r2 = r_{grid}^2$
    *   $x_{min} = \max(0, \lfloor c_x - r_{grid} \rfloor)$
    *   $x_{max} = \min(\text{grid\_dim}-1, \lceil c_x + r_{grid} \rceil)$
    *   Repeat for $y$ and $z$ axes [S1, S2, unofficial].
4.  **Iteration and Test**: Iterate $x$ from $x_{min}$ to $x_{max}$, $y$ from $y_{min}$ to $y_{max}$, $z$ from $z_{min}$ to $z_{max}$ [S1, S2, unofficial].
    *   Calculate squared distance from voxel center to sphere center:
        $$d^2 = (x + 0.5 - c_x)^2 + (y + 0.5 - c_y)^2 + (z + 0.5 - c_z)^2$$
        [S2, unofficial]
    *   If $d^2 \le r2$ and the voxel is solid (`grid[x,y,z] != 0`):
        *   Remove the voxel: `grid[x,y,z] = 0` [S1, S2, unofficial].
        *   Increment `removed_count`.
        *   Record world position for gib spawning (if required) [S2, unofficial].
5.  **Post-Processing**:
    *   Update `solid_count` by subtracting `removed_count` [S2, unofficial].
    *   Set `dirty` flag to true if `removed_count > 0` [S2, unofficial].
    *   Return `removed_count` and `gib_positions` [S1, S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sphere_carving(grid, grid_dim, vox_size, center_world, radius_world, origin_world, rotation_world_inv):
    """
    Removes solid voxels within a sphere from a voxel grid.
    grid: 3D array or SVO accessor
    grid_dim: int, size of the grid/chunk
    vox_size: float, world units per voxel
    center_world: Vec3, world space center of carve
    radius_world: float, world space radius
    origin_world: Vec3, world space origin of the grid/object
    rotation_world_inv: Quat, inverse rotation of the object (if needed)
    """
    # 1. Transform center to local grid coordinates
    # Subtract origin
    local_center = center_world - origin_world
    # Apply inverse rotation (if rotation is not identity/axis-aligned)
    # Note: Sources indicate rotation is often unused for axis-aligned SVOs [S2, unofficial]
    # local_center = rotation_world_inv * local_center 
    
    # Convert to voxel coordinates
    # Add half extent to center voxel, divide by voxel size
    # Assuming grid is centered at origin or offset is handled by 'origin'
    # Standard mapping: voxel_index = (world_pos - min_world) / vox_size
    # Here we assume local_center is relative to grid center or min, 
    # based on source [S2, unofficial]: "adding half the grid extent and dividing by vox_size"
    # Let's assume local_center is already relative to grid min or center appropriately.
    # Source [S2, unofficial] implies: cx = (local_x + extent/2) / vox_size
    # We need extent. If grid_dim is size, and origin is center, extent might be implicit.
    # Simplified based on [S2, unofficial]:
    cx = (local_center.x + (grid_dim * vox_size) / 2.0) / vox_size
    cy = (local_center.y + (grid_dim * vox_size) / 2.0) / vox_size
    cz = (local_center.z + (grid_dim * vox_size) / 2.0) / vox_size

    # 2. Convert radius to grid units
    r_grid = radius_world / vox_size
    r2 = r_grid * r_grid

    # 3. Calculate bounding box
    xmin = max(0, floor(cx - r_grid))
    xmax = min(grid_dim - 1, ceil(cx + r_grid))
    ymin = max(0, floor(cy - r_grid))
    ymax = min(grid_dim - 1, ceil(cy + r_grid))
    zmin = max(0, floor(cz - r_grid))
    zmax = min(grid_dim - 1, ceil(cz + r_grid))

    removed_count = 0
    gib_positions = []

    # 4. Iterate and remove
    for z in range(zmin, zmax + 1):
        for y in range(ymin, ymax + 1):
            for x in range(xmin, xmax + 1):
                # Calculate squared distance from voxel center to sphere center
                # Voxel center in grid units is (x + 0.5, y + 0.5, z + 0.5)
                dx = (x + 0.5) - cx
                dy = (y + 0.5) - cy
                dz = (z + 0.5) - cz
                dist_sq = dx*dx + dy*dy + dz*dz

                if dist_sq <= r2:
                    # Check if solid
                    if grid.get(x, y, z) != 0:
                        grid.set(x, y, z, 0)
                        removed_count += 1
                        
                        # Record world position for gibs
                        # World pos = (voxel_index + 0.5) * vox_size + grid_min_world
                        # Assuming grid_min_world is origin_world - (extent/2) if origin is center
                        # Or simply use local_center logic reversed
                        world_x = (x + 0.5) * vox_size + (origin_world.x - (grid_dim * vox_size) / 2.0)
                        world_y = (y + 0.5) * vox_size + (origin_world.y - (grid_dim * vox_size) / 2.0)
                        world_z = (z + 0.5) * vox_size + (origin_world.z - (grid_dim * vox_size) / 2.0)
                        gib_positions.append(Vec3(world_x, world_y, world_z))

    # 5. Post-processing
    # Update solid_count and dirty flag in the owning object
    # This is handled by the caller or object wrapper [S2, unofficial]
    
    return removed_count, gib_positions
```

### Parameters
*   `radius`: Typical values depend on gameplay scale; must be positive [S1, S2, unofficial].
*   `vox_size`: Determines resolution; smaller values increase precision but increase iteration count [S2, unofficial].

### Edge Cases and Pitfalls
*   **Precision**: Floating point errors in distance calculation can cause voxels on the boundary to be inconsistently removed. Using squared distance avoids square roots but requires careful handling of $r2$ [S1, S2, unofficial].
*   **Boundaries**: The bounding box must be strictly clamped to the grid dimensions to avoid out-of-bounds access [S1, S2, unofficial].
*   **SVO Traversal**: In SVO implementations, the algorithm must traverse only existing nodes. Empty nodes are skipped, but the bounding box logic still applies to the leaf level or the specific chunk being carved [S2, unofficial].
*   **Degenerate Input**: Radius of 0 or negative radius should return 0 removed voxels [G].

## Subtopics

### [Voxel Deformation with Spherical Falloff](4413-voxel-deformation-with-spherical-falloff.md)
Voxel Deformation with Spherical Falloff modifies scalar voxel values within a spherical region around a 3D hit point to add or remove terrain mass, enabling real-time digging and building operations. It sets itself apart from [Sphere Carving](2377-sphere-carving.md) by using a continuous soft falloff function to create smooth edges rather than binary inclusion/exclusion, and by supporting additive (build) as well as subtractive (dig) operations.

### Bounding Ellipsoids from SDFs
Generating a bounding volume as a union of ellipsoids by replacing the convex hull generation step with minimum-volume enclosing ellipsoid generation on the point set P [S3]. The signed distance function for an ellipsoid $E$ is $d(\mathbf{p}, \mathcal{E}) = a (\|\mathbf{A}(\mathbf{p} - \mathbf{c})\| - 1)$, where $\mathbf{A} = \mathbf{D}(1/a, 1/b, 1/c) \mathbf{R}^t$, with $c$ as center, $R$ as rotation matrix, and $D$ as diagonal matrix [S3]. Evaluation requires approximately 4 scalar products and one square root [S3]. Ellipsoids are more compact in memory and computationally more efficient than convex polyhedra, though potentially more conservative depending on axis lengths [S3].

### Voxel Edit Brush Operations
Four brush operations (Place, Carve, Fill, Paint) applied over a world-space spherical radius on an unbounded voxel grid [S4, unofficial]. Voxels are addressed using unbounded world-grid coordinates rather than chunk-local indices to ensure consistent behavior across chunk boundaries [S4, unofficial]. Edits are accumulated into a stroke across frames, and a single `VoxelEditCommand` is pushed on release [S4, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(r^3)$ where $r$ is the radius in voxels, as it iterates over the bounding box of the sphere [S1, S2, unofficial].
*   **Memory Complexity**: $O(1)$ auxiliary space for the iteration, plus $O(k)$ for storing gib positions if required, where $k$ is the number of removed voxels [S1, S2, unofficial].
*   **Performance**: Performance scales linearly with the volume of the carving sphere. For SVOs, performance is better than dense grids because empty space is skipped, but the bounding box traversal still requires checking node existence [S2, unofficial].
*   **Strengths**: Simple to implement, efficient for small to medium radius carves, supports dynamic destruction [S1, S2, unofficial].
*   **Limitations**: Not suitable for very large radius carves due to $O(r^3)$ complexity; binary removal lacks smoothness compared to falloff methods [S1, S2, unofficial].
*   **When to use**: Destructible terrain, breaking objects, real-time editing [S1, S2, S4, unofficial].
*   **When not to use**: When smooth erosion or soft deformation is required (use Voxel Deformation with Spherical Falloff) [S1, S2, unofficial].

## Background
Sphere Carving is a fundamental technique in voxel-based games for implementing destructible environments. It is widely used in games like Minecraft and various voxel editors. The method is often implemented as a modification to Sparse Voxel Octrees (SVO) to handle large-scale destruction efficiently [S2, unofficial]. The concept of using spherical kernels for voxel manipulation is also related to Signed Distance Field (SDF) carving, where spheres are used to approximate complex shapes or bounding volumes [S3].

## Sources
[S1] m2-md/voxel-greedy-meshing-typescript, https://github.com/m2-md/voxel-greedy-meshing-typescript, fetched 2026-10-26 (unofficial)
[S2] elyssov/prometheus-engine, https://github.com/elyssov/prometheus-engine, fetched 2026-10-28 (unofficial)
[S3] Sphere Carving: Bounding Volumes for Signed Distance Fields, https://doi.org/10.1145/3730845, fetched 2026-10-27
[S4] feat(terrain): DDA voxel raycast + editing brushes with undo/redo, https://github.com/drsnuggles8/OloEngineBase/pull/973, fetched 2026-10-25 (unofficial)
