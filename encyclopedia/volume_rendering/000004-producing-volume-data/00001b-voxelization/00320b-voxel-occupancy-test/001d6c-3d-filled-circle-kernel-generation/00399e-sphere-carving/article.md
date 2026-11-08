# Sphere Carving

Also called: Carve Sphere, SVO Sphere Carving, SVO modification, Voxel Destruction via SVO, Voxel Sphere Removal, voxel removal

## Overview
Sphere Carving is a voxelization post-processing method that removes solid voxels within a spherical radius from a specified center point in a voxel chunk or Sparse Voxel Octree (SVO) [S1, S2, unofficial]. It solves the problem of creating destructible terrain or breakable objects by efficiently identifying and nullifying voxels intersecting a spherical volume [S1, unofficial]. It sets itself apart from its parent [3D Filled Circle Kernel Generation](1520-3d-filled-circle-kernel-generation.md) by operating on existing occupancy data to remove voxels rather than generating a static binary kernel, and from [3D Binary Sphere Kernel Generation](4654-3d-binary-sphere-kernel-generation.md) by focusing on the destructive modification of a scene graph rather than kernel synthesis [S1, S2, unofficial].

## How it works
**Inputs:**
- `chunk`: The voxel data structure to modify (e.g., a 3D array or SVO node) [S1, S2, unofficial].
- `center`: The world-space or local-space center of the carving sphere [S1, S2, unofficial].
- `radius`: The radius of the sphere in world units [S1, S2, unofficial].

**Outputs:**
- Modified `chunk` with voxels inside the sphere set to 0 (air/empty) [S1, S2, unofficial].
- `removed_count`: The number of solid voxels that were removed [S1, S2, unofficial].
- `gib_positions`: (Optional) A list of world positions of the removed voxels, used for spawning particle effects or physics bodies [S2, unofficial].

**Data Structures:**
- **Voxel Chunk/Grid:** A 3D array or sparse structure indexed by `(x, y, z)`. Values are typically integers where `0` represents air/empty and non-zero values represent solid material [S1, S2, unofficial].
- **VoxObject (SVO Context):** A struct containing:
  - `origin: Vec3`: World center of the object [S2, unofficial].
  - `rotation: Quat`: Orientation (unused for axis-aligned carving) [S2, unofficial].
  - `vox_size: f32`: World units per voxel [S2, unofficial].
  - `grid_dim: usize`: Power-of-two grid size [S2, unofficial].
  - `svo: SVO`: The sparse octree data [S2, unofficial].
  - `solid_count: u32`: Current count of solid voxels [S2, unofficial].
  - `dirty: bool`: Flag indicating if the mesh needs rebuilding [S2, unofficial].

**Steps:**
1. **Coordinate Transformation:** If the input `center` is in world space and the voxel data is in local space, transform the center to local coordinates. This involves subtracting the object's origin and applying the inverse rotation [S2, unofficial].
2. **Grid Conversion:** Convert the local center coordinates `(cx, cy, cz)` to voxel grid indices. This is done by adding half the grid extent (if using center-based indexing) and dividing by `vox_size` [S2, unofficial].
3. **Radius Conversion:** Calculate the radius in grid units: `r_grid = radius / vox_size` [S2, unofficial]. Compute the squared radius: `r2 = r_grid * r_grid` [S1, S2, unofficial].
4. **Bounding Box Calculation:** Determine the axis-aligned bounding box (AABB) of the sphere in grid indices to minimize checks. For each axis `v` (x, y, z):
   - `lo(v) = max(0, floor(center_v - r_grid))`
   - `hi(v) = min(grid_dim - 1, ceil(center_v + r_grid))`
   Clamp these bounds to the grid dimensions `[0, grid_dim - 1]` [S1, S2, unofficial].
5. **Voxel Iteration:** Iterate through all voxel indices `(x, y, z)` within the bounding box [S1, S2, unofficial].
6. **Distance Check:** For each voxel, calculate the squared Euclidean distance from the voxel center to the sphere center. The voxel center in grid units is typically `(x + 0.5, y + 0.5, z + 0.5)` [S2, unofficial].
   - `dx = (x + 0.5) - cx`
   - `dy = (y + 0.5) - cy`
   - `dz = (z + 0.5) - cz`
   - `dist_sq = dx*dx + dy*dy + dz*dz`
   - If `dist_sq > r2`, continue to the next voxel [S1, S2, unofficial].
7. **Removal:** If the voxel is solid (value != 0):
   - Set the voxel value to 0 (`chunk.set(x, y, z, 0)`) [S1, S2, unofficial].
   - Increment `removed_count` [S1, S2, unofficial].
   - Calculate the world position of the removed voxel for `gib_positions` (if required) [S2, unofficial].
8. **State Update:** After the loop, if `removed_count > 0`:
   - Update `solid_count` by subtracting `removed_count` [S2, unofficial].
   - Mark the object/mesh as `dirty` to trigger a rebuild [S2, unofficial].
   - Return `removed_count` and `gib_positions` [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sphere_carving(chunk, center_world, radius_world, vox_size, grid_dim, origin_world, inverse_rotation):
    # 1. Transform center to local space
    local_center = inverse_rotation * (center_world - origin_world)
    
    # 2. Convert to grid coordinates
    # Assuming center-based indexing: add 0.5 if grid indices are cell corners, 
    # or adjust based on specific voxel coordinate system.
    # Standard approach: voxel center at index i is i + 0.5
    cx = local_center.x / vox_size
    cy = local_center.y / vox_size
    cz = local_center.z / vox_size
    
    # 3. Calculate grid radius
    r_grid = radius_world / vox_size
    r2 = r_grid * r_grid
    
    # 4. Calculate bounding box
    lo_x = max(0, floor(cx - r_grid))
    hi_x = min(grid_dim - 1, ceil(cx + r_grid))
    lo_y = max(0, floor(cy - r_grid))
    hi_y = min(grid_dim - 1, ceil(cy + r_grid))
    lo_z = max(0, floor(cz - r_grid))
    hi_z = min(grid_dim - 1, ceil(cz + r_grid))
    
    removed_count = 0
    gib_positions = []
    
    # 5. Iterate through bounding box
    for z in range(lo_z, hi_z + 1):
        for y in range(lo_y, hi_y + 1):
            for x in range(lo_x, hi_x + 1):
                # 6. Distance check
                # Voxel center in local grid units
                vx = x + 0.5
                vy = y + 0.5
                vz = z + 0.5
                
                dx = vx - cx
                dy = vy - cy
                dz = vz - cz
                
                dist_sq = dx*dx + dy*dy + dz*dz
                
                if dist_sq <= r2:
                    # 7. Removal
                    if chunk.get(x, y, z) != 0:
                        chunk.set(x, y, z, 0)
                        removed_count += 1
                        
                        # Calculate world position for gib
                        world_pos = origin_world + (inverse_rotation.inverse() * Vec3(x * vox_size, y * vox_size, z * vox_size))
                        gib_positions.append(world_pos)
    
    # 8. State update (example for SVO object)
    # object.solid_count -= removed_count
    # object.dirty = removed_count > 0
    
    return removed_count, gib_positions
```

**Parameters:**
- `radius`: Typical values depend on the scale of the voxel grid. For a 128x128x128 chunk with 1 unit voxels, radii might range from 1 to 64 [S1, S2, unofficial].
- `vox_size`: Determines the resolution. Smaller `vox_size` increases accuracy but increases computational cost [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Boundary Clamping:** The bounding box must be strictly clamped to the grid dimensions to avoid out-of-bounds memory access [S1, S2, unofficial].
- **Precision:** Using squared distances avoids expensive square root operations during the inner loop, improving performance [S1, S2, unofficial].
- **Degenerate Input:** If `radius` is 0, no voxels are removed. If the center is far outside the grid, the bounding box calculation should result in an empty range [S1, S2, unofficial].

## Subtopics

### [Voxel Deformation with Spherical Falloff](4413-voxel-deformation-with-spherical-falloff.md)
Voxel Deformation with Spherical Falloff modifies scalar voxel values within a spherical region around a 3D hit point to add or remove terrain mass, enabling real-time digging and building operations. It sets itself apart from its parent [Sphere Carving](2377-sphere-carving.md) by using a continuous soft falloff function to create smooth edges rather than binary inclusion/exclusion, and by supporting additive (build) as well as subtractive (dig) operations.

### [Procedural Sphere Placement](8171-procedural-sphere-placement.md)
Procedural Sphere Placement populates a 3D voxel grid with spherical voids (density 0) inside a background field (density 255), using pre-sampled 2D coordinates and random radii. It differs from its parent [Sphere Carving](2377-sphere-carving.md) by automating the generation of multiple, randomly sized and positioned spheres rather than carving a single defined sphere, and from [Voxel Deformation with Spherical Falloff](4413-voxel-deformation-with-spherical-falloff.md) by producing hard-edged binary voids rather than smooth density gradients.

### Bounding Ellipsoids from SDFs
Generating a bounding volume as a union of ellipsoids by replacing the convex hull generation step with minimum-volume enclosing ellipsoid generation on the point set P [S4]. The signed distance function for an ellipsoid E is defined as $d(\mathbf{p}, \mathcal{E}) = a (\|\mathbf{A}(\mathbf{p} - \mathbf{c})\| - 1)$, where $\mathbf{A} = \mathbf{D}(1/a, 1/b, 1/c) \mathbf{R}^t$, with $\mathbf{c}$ as the center, $\mathbf{R}$ as the rotation matrix, and $\mathbf{D}$ as the diagonal matrix [S4]. Evaluation requires approximately 4 scalar products and one square root [S4]. Ellipsoids are more compact in memory and computationally more efficient than convex polyhedra, though potentially more conservative depending on axis lengths [S4].

### Voxel Edit Brush Operations
Four brush operations (Place, Carve, Fill, Paint) applied over a world-space spherical radius on an unbounded voxel grid [S5, unofficial]. Voxels are addressed using unbounded world-grid coordinates rather than chunk-local indices to ensure consistent behavior across chunk boundaries [S5, unofficial]. Edits are accumulated into a stroke across frames, and a single `VoxelEditCommand` is pushed on release [S5, unofficial].

### Voxel Annihilation
Removes all solid voxels from the entire SVO grid, used when a hit's effective power is overwhelmingly larger than the object's hp [S2, unofficial]. It iterates through all grid indices $(x, y, z)$ from 0 to `grid_dim`, removes solid voxels, calculates their world positions for gib spawning, sets `solid_count` to 0, and marks the object as `dirty` [S2, unofficial].

### Collision and Destructibility with Empty Zones
Method to integrate Kernel-Reflection Sequences (KRS) into collision detection by creating empty zones via unbounding spheres and substituting them with physically simulated kernel instances [S6, unofficial]. It creates empty zones by specifying unbounding spheres over subsets of the KRS, skips these regions during sphere tracing, substitutes the subset with physically simulated instances, and applies physical simulation to them [S6, unofficial]. A pitfall is that KRS is not locally controllable [S6, unofficial]. Example: A tree branch breaks by covering the complete tree with an empty sphere and building an identical tree of rigid body branches and leaves independently [S6, unofficial].

## Operating characteristics
- **Time Complexity:** $O(r^3)$ where $r$ is the radius in grid units, as it iterates over the bounding box of the sphere [S1, S2, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space for the algorithm itself, excluding the storage for `gib_positions` which is $O(k)$ where $k$ is the number of removed voxels [S1, S2, unofficial].
- **Performance:** Efficient for small to medium radii. For large radii, the number of voxels checked grows cubically [S1, S2, unofficial].
- **Strengths:** Simple to implement, fast for localized destruction, works with any voxel representation [S1, S2, unofficial].
- **Limitations:** Does not handle complex shapes or non-spherical destruction patterns. Binary removal may lead to jagged edges if not combined with smoothing techniques [S1, S2, unofficial].
- **When to use:** Destructible terrain, breakable objects, explosion effects [S1, S2, unofficial].
- **When not to use:** When smooth deformation is required (use [Voxel Deformation with Spherical Falloff](4413-voxel-deformation-with-spherical-falloff.md)) or when procedural generation of multiple voids is needed (use [Procedural Sphere Placement](8171-procedural-sphere-placement.md)) [S1, S2, unofficial].

## Background
Sphere Carving originates from implementations in voxel-based games and engines, particularly those using Sparse Voxel Octrees (SVO) for destructible environments [S1, S2, unofficial]. It is a fundamental technique for implementing player-driven terrain modification and object destruction [S1, S2, unofficial]. The method is often associated with real-time voxel editors and games like Minecraft clones or voxel-based RPGs [S1, S2, unofficial].

## Sources
[S1] m2-md/voxel-greedy-meshing-typescript, https://github.com/m2-md/voxel-greedy-meshing-typescript, fetched 2026-10-26 (unofficial)
[S2] elyssov/prometheus-engine, https://github.com/elyssov/prometheus-engine, fetched 2026-10-28 (unofficial)
[S3] Enhanced Sphere Tracing | Request PDF - ResearchGate, https://www.researchgate.net/publication/265020476_Enhanced_Sphere_Tracing, fetched 2026-11-03 (abstract only, unofficial)
[S4] Sphere Carving: Bounding Volumes for Signed Distance Fields, https://doi.org/10.1145/3730845, fetched 2026-10-27
[S5] feat(terrain): DDA voxel raycast + editing brushes with undo/redo, https://github.com/drsnuggles8/OloEngineBase/pull/973, fetched 2026-10-25 (unofficial)
[S6] Kernel-Reflection Sequences, https://doi.org/10.2312/stag.20161364, fetched 2026-10-26 (unofficial)
