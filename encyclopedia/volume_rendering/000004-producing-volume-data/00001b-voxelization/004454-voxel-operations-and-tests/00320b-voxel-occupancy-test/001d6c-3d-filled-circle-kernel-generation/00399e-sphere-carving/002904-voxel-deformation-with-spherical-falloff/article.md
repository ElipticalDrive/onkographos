# Voxel Deformation with Spherical Falloff

Also called: Spherical Voxel Deformation, Voxel Falloff Modification

## Overview
Voxel Deformation with Spherical Falloff modifies scalar voxel values within a spherical region around a 3D hit point to add or remove terrain mass, enabling real-time digging and building operations [S1, S2, unofficial]. It sets itself apart from its parent [Sphere Carving](2377-sphere-carving.md) by using a continuous soft falloff function to create smooth edges rather than binary inclusion/exclusion, and by supporting additive (build) as well as subtractive (dig) operations [S1, S2, unofficial].

## How it works
**Inputs:**
- `playerCamera`: Camera object providing position and forward direction [S1, S2, unofficial].
- `hitPoint`: World position where the deformation occurs, obtained via raycast [S1, S2, unofficial].
- `radius`: Brush radius in meters [S2, unofficial].
- `deformSpeed` (S1) or `digSpeed`/`buildSpeed` (S2): Rate of density change per second at the center of the brush [S1, S2, unofficial].
- `buildModifier`: Integer `1` for adding mass, `-1` for removing mass [S1, unofficial].
- `voxelSize`: Size of one voxel in meters [S2, unofficial].
- `densityField`: Array of scalar values representing the terrain [S1, S2, unofficial].
- `chunkSize`: Size of a terrain chunk in voxels [S2, unofficial].

**Outputs:**
- Updated `densityField` values within the affected region [S1, S2, unofficial].
- List of "dirty" chunks marked for re-meshing [S2, unofficial].

**Data Structures:**
- **Density Field:** A 3D grid of scalar values [S1, S2, unofficial]. In [S1, unofficial], values are 8-bit unsigned integers (`byte`) in range `[0, 255]`, where 0 is air and 255 is solid rock [S1, unofficial]. In [S2, unofficial], values are `float` in range `[0, 1]` [S2, unofficial].
- **Chunk:** A sub-volume of the world defined by `chunkSize` [S2, unofficial]. Each chunk manages its own mesh and collision data [S2, unofficial].
- **Bounds:** Axis-aligned bounding box (AABB) in voxel coordinates defining the region to iterate [S1, S2, unofficial].

**Steps:**
1. **Raycast:** Cast a ray from `playerCamera.position` along `playerCamera.forward` with a maximum distance `maxReachDistance` (or `range`) [S1, S2, unofficial]. If no hit is detected, abort [S1, S2, unofficial].
2. **Hit Point Adjustment (Build Only):** If the operation is a build, offset the hit point slightly into the surface to ensure new material attaches correctly: `hitPoint -= rayDirection * radius * 0.3f` [S2, unofficial].
3. **Coordinate Conversion:** Convert the world-space `hitPoint` to local voxel coordinates: `local = (hitPoint - worldOrigin) / voxelSize` [S2, unofficial]. (Note: [S1, unofficial] assumes direct conversion to integer voxel coordinates `(hitX, hitY, hitZ)`).
4. **Bounding Box Calculation:** Calculate the integer bounding box of the sphere in voxel coordinates: `minX = Floor(local.x - radius_voxels)`, `maxX = Ceil(local.x + radius_voxels)`, etc., clamped to the world boundaries [S2, unofficial]. [S1, unofficial] uses `intRange = ceil(deformRange)` and defines bounds as `[hitX - intRange, hitX + intRange]` etc.
5. **Chunk Identification:** Identify all chunks that overlap with this bounding box [S2, unofficial]. Mark them as "dirty" [S2, unofficial].
6. **Voxel Iteration:** Iterate over all voxels `(x, y, z)` within the bounding box [S1, S2, unofficial]. [S2, unofficial] specifies skipping border voxels if they are outside the world or protected [S2, unofficial].
7. **Distance Calculation:** For each voxel, calculate the Euclidean distance `d` from the voxel center to the brush center `local`: `d = sqrt((x - local.x)^2 + (y - local.y)^2 + (z - local.z)^2)` [S1, S2, unofficial].
8. **Falloff Calculation:** [S1, unofficial] uses an inverse distance falloff: `modificationAmount = deformSpeed / d * buildModifier` [S1, unofficial]. [S2, unofficial] uses a SmoothStep falloff: `falloff = SmoothStep(1, 0, dist / radius_voxels)` [S2, unofficial].
9. **Density Update:** [S1, unofficial] (Byte-based): `v_new = clamp((v_old / 255 - modificationAmount) * 255, 0, 255)` [S1, unofficial]. [S2, unofficial] (Float-based): `density[i] = Clamp01(density[i] + delta * falloff)`, where `delta = speed * deltaTime` [S2, unofficial].
10. **Re-meshing:** In `LateUpdate`, iterate over all "dirty" chunks and call `Rebuild()` on each, then clear the dirty set [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def raycast(origin, direction, max_distance):
    """Stub for physics raycast; returns hit dict with 'point' or None."""
    # Implementation depends on engine; returns hit or None
    pass

def get_chunk_id(x, y, z, chunk_size):
    """Returns chunk coordinates for a given voxel."""
    return (x // chunk_size, y // chunk_size, z // chunk_size)

def smoothstep(edge0, edge1, x):
    """Standard smooth interpolation."""
    t = max(0.0, min(1.0, (x - edge0) / (edge1 - edge0)))
    return t * t * (3.0 - 2.0 * t)

def deform_voxel_terrain(camera, terrain, radius, speed, is_build, delta_time):
    # 1. Raycast
    ray_origin = camera.position
    ray_direction = camera.forward
    hit = raycast(ray_origin, ray_direction, terrain.max_reach)
    if hit is None:
        return

    hit_point = hit.point

    # 2. Adjust hit point for build
    if is_build:
        hit_point = hit_point - ray_direction * radius * 0.3

    # 3. Convert to voxel coordinates
    local_center = (hit_point - terrain.origin) / terrain.voxel_size

    # 4. Calculate bounding box (convert radius to voxel units for scale consistency)
    radius_voxels = radius / terrain.voxel_size
    min_x = int(math.floor(local_center.x - radius_voxels))
    max_x = int(math.ceil(local_center.x + radius_voxels))
    min_y = int(math.floor(local_center.y - radius_voxels))
    max_y = int(math.ceil(local_center.y + radius_voxels))
    min_z = int(math.floor(local_center.z - radius_voxels))
    max_z = int(math.ceil(local_center.z + radius_voxels))

    # Clamp to world bounds
    min_x = max(min_x, 0); max_x = min(max_x, terrain.width - 1)
    min_y = max(min_y, 0); max_y = min(max_y, terrain.height - 1)
    min_z = max(min_z, 0); max_z = min(max_z, terrain.depth - 1)

    # 5. Identify and mark dirty chunks
    dirty_chunks = set()
    for x in range(min_x, max_x + 1):
        for y in range(min_y, max_y + 1):
            for z in range(min_z, max_z + 1):
                dirty_chunks.add(get_chunk_id(x, y, z, terrain.chunk_size))

    # 6. Iterate voxels and update
    for x in range(min_x, max_x + 1):
        for y in range(min_y, max_y + 1):
            for z in range(min_z, max_z + 1):
                dx = x - local_center.x
                dy = y - local_center.y
                dz = z - local_center.z
                dist = math.sqrt(dx*dx + dy*dy + dz*dz)
                
                # Skip if outside radius
                if dist > radius_voxels:
                    continue

                # Calculate falloff (SmoothStep variant [S2, unofficial])
                falloff = smoothstep(1.0, 0.0, dist / radius_voxels)
                
                # Calculate delta
                delta = speed * delta_time * falloff
                modifier = 1.0 if is_build else -1.0

                # Get old value
                old_val = terrain.density_field[x, y, z]

                # Update value
                new_val = old_val + delta * modifier

                # Clamp to [0.0, 1.0]
                terrain.density_field[x, y, z] = max(0.0, min(1.0, new_val))

    # 7. Mark chunks dirty
    for chunk_id in dirty_chunks:
        terrain.mark_chunk_dirty(chunk_id)

def late_update_terrain(terrain):
    dirty_chunks = terrain.get_dirty_chunks()
    for chunk_id in dirty_chunks:
        terrain.rebuild_chunk(chunk_id)
    terrain.clear_dirty_chunks()
```

**Parameters:**
- `radius`: Brush radius in meters. Typical value: 2.5f. Range: 1f to 6f [S2, unofficial].
- `digSpeed`: Density removed per second at the center. Typical value: 4f [S2, unofficial].
- `buildSpeed`: Density added per second at the center. Typical value: 3f [S2, unofficial].
- `smoothSpeed`: Smoothing speed. Typical value: 6f [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** If the raycast hit point exactly matches a voxel center, `distance` becomes 0, causing infinite `modificationAmount` in inverse-distance falloff [S1, unofficial]. A guard such as `if dist == 0: dist = 1e-6` is required [G].
- **Precision:** Using `byte` values (0-255) as in [S1, unofficial] can lead to quantization artifacts compared to `float` (0-1) as in [S2, unofficial] [S1, S2, unofficial].
- **Boundary Conditions:** Voxels at the world boundary must be clamped to prevent out-of-bounds array access [S2, unofficial].
- **Scale Mismatch:** The brush radius is provided in meters, but bounding box calculations require voxel units; failing to divide by `voxelSize` will produce incorrect deformation volumes [G].

## Subtopics

### Local Area Smoothing Brush
A variant that locally blurs the density field within a spherical region to round off jagged edges created by deformation [S2, unofficial]. It differs from standard deformation by averaging density values with neighbors rather than adding/removing mass [S2, unofficial].
- **Definition:** Locally blurs the density field within a spherical region to round off jagged edges, using a snapshot of the field to ensure consistent averaging [S2, unofficial].
- **Steps:**
  1. Calculate the bounding box of the brush area, expanded by 1 voxel in all directions to include neighbors: `sMinX = Max(minX - 1, 0)`, `sMaxX = Min(maxX + 1, pointCount.x - 1)`, etc. [S2, unofficial].
  2. Create a snapshot array of the expanded region and copy the current density values into it [S2, unofficial].
  3. Iterate through the original brush bounding box (not the expanded one). Skip border voxels and voxels outside the radius [S2, unofficial].
  4. Calculate the average density from the snapshot using the voxel and its 6 neighbors, clamping coordinates to the snapshot bounds: `average = (Snap(x,y,z) + Snap(x+1,y,z) + Snap(x-1,y,z) + Snap(x,y+1,z) + Snap(x,y-1,z) + Snap(x,y,z+1) + Snap(x,y,z-1)) / 7.0f` [S2, unofficial].
  5. Calculate the falloff factor: `falloff = SmoothStep(1, 0, dist / r) * strength` [S2, unofficial].
  6. Interpolate the density towards the average: `density[i] = Lerp(density[i], average, falloff)` [S2, unofficial].
- **When to choose:** Use after digging or building to smooth out stair-stepping artifacts and create natural-looking terrain transitions [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(R^3)$ per deformation call, where $R$ is `deformRange` in voxel units, due to iterating over the full bounding box of the spherical region [S1, unofficial].
- **Memory Complexity:** $O(R^3)$ for the bounding box iteration. The Local Area Smoothing Brush variant requires an additional $O((R+1)^3)$ memory for the snapshot array [S2, unofficial].
- **Measured Performance:** Real-time deformation is achieved using three brushes (Dig, Build, Smooth) with soft spherical falloffs [S2, unofficial]. Performance scales with the cube of the brush radius [S1, unofficial].
- **Strengths:** Smooth edges via falloff functions; supports both addition and subtraction; real-time capable [S1, S2, unofficial].
- **Limitations:** $O(R^3)$ complexity can be expensive for large brushes; inverse-distance falloff requires division-by-zero guards [S1, unofficial].
- **When to use:** Destructible terrain scenarios requiring smooth, natural-looking modifications [S1, S2, unofficial].
- **When not to use:** When strict binary voxel occupancy is required (use [Sphere Carving](2377-sphere-carving.md) instead) [S1, unofficial].

## Background
Voxel Deformation with Spherical Falloff is used in real-time voxel terrain editors and games, particularly those utilizing Marching Cubes for mesh generation [S1, S2, unofficial]. It builds on [Sphere Carving](2377-sphere-carving.md) by adding continuous density modification [S1, S2, unofficial]. The method is implemented in Unity-based voxel terrain projects such as Eldemarkki/Marching-Cubes-Terrain and Rabidus4k/UnityVoxelTerrain [S1, S2, unofficial].

## Sources
[S1] Eldemarkki/Marching-Cubes-Terrain, https://github.com/Eldemarkki/Marching-Cubes-Terrain, fetched 2026-10-26 (unofficial)
[S2] Rabidus4k/UnityVoxelTerrain, https://github.com/Rabidus4k/UnityVoxelTerrain, fetched 2026-10-28 (unofficial)
