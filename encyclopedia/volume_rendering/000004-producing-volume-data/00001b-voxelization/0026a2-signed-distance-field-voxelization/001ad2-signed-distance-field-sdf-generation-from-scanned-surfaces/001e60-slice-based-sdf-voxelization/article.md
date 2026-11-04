# Slice-based SDF Voxelization
Also called: Parity-based Voxelization, Ray-cast Voxelization

## Overview
Slice-based SDF Voxelization computes a Signed Distance Field (SDF) for a triangulated surface mesh by iterating over a 3D grid, determining inside/outside status via ray-casting parity on 2D slices, and sampling the exact mesh distance at each voxel center [S1, unofficial]. It differs from [Signed Distance Field from Mesh](1327-signed-distance-field-from-mesh.md) by explicitly using 2D slice intersections to determine topological containment (inside/outside) rather than relying on global distance field propagation or hierarchical structures for sign determination [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangulated surface mesh (e.g., `.stl` format) [S1, unofficial].
*   **Parameters:**
    *   `resolution`: Grid spacing in world units. Must be a finite number $> 10^{-6}$ [S1, unofficial].
    *   `narrow_band`: Optional non-negative finite float specifying the maximum absolute SDF value for a voxel to be retained. Defaults to keeping all inside voxels if omitted [S1, unofficial].
    *   `phase_sphere`: Optional array `[cx, cy, cz, r]` defining a sphere. Voxels inside this sphere are marked with `phase = 1` [S1, unofficial].
*   **Output:** A list of `ParticleData` records containing voxel coordinates, SDF, phase, and auxiliary fields [S1, unofficial].

**Data Structures:**
*   `ParticleData`: A struct stored in a contiguous `Vec` or parallel arrays with the following fields [S1, unofficial]:
    *   `x: f32`, `y: f32`, `z: f32`: Voxel center coordinates.
    *   `sdf: f32`: Signed distance value.
    *   `phase: u32` (or `i32`): Phase indicator (0 or 1).
    *   `label_id: u32`: Label identifier (zero-initialized).
    *   `fiber_x: f32`, `fiber_y: f32`: Auxiliary fields (zero-initialized).

**Algorithm Steps:**
1.  **Validation:** Check `resolution` and `narrow_band`. If `resolution` $\le 10^{-6}$, negative, NaN, or Infinity, abort. If `narrow_band` is provided and is negative, NaN, Infinity, or NegInfinity, abort [S1, unofficial].
2.  **Grid Bounds:** Determine the bounding box of the mesh. Define grid dimensions $N_x, N_y, N_z$ based on the bounding box and `resolution` [S1, unofficial].
3.  **Slice Processing (Parallel over Y, Z):** For each fixed $y$ and $z$ coordinate in the grid:
    a.  **Ray Casting:** Cast a ray along the +X direction from $x_{start} = bounds_{min}.x - 1.0$ to $x_{end} = bounds_{max}.x + 1.0$ at the current $(y, z)$ plane [S1, unofficial].
    b.  **Intersection Collection:** Collect all intersection X-coordinates into a list `hit_xs`. To handle grazing intersections or precision issues, the ray origin is shifted by a small epsilon ($10^{-4}$) after each hit to find subsequent intersections [S1, unofficial]. Limit the number of hits to 100,000 to prevent excessive memory usage [S1, unofficial].
    c.  **Sorting:** Sort `hit_xs` in ascending order to ensure consistent parity calculation [S1, unofficial].
4.  **Voxel Evaluation (Parallel over X):** For each voxel index $i_x$ in the current slice:
    a.  **Coordinate Calculation:** Compute the voxel center $x = bounds_{min}.x + (i_x \cdot resolution) + (resolution \cdot 0.5)$ [S1, unofficial].
    b.  **Inside/Outside Check:** Calculate the number of intersections to the right of the voxel center using binary search (partition point) on the sorted `hit_xs` list:
        $$ \text{intersections\_to\_right} = \text{hit\_xs.len()} - \text{partition\_point}(|hx| \ hx \le x) $$
        Determine inside status:
        $$ \text{is\_inside} = (\text{intersections\_to\_right} \pmod 2 \neq 0) $$
        [S1, unofficial]
    c.  **Distance Calculation:** Compute the Euclidean distance $d$ from the voxel center to the mesh surface using a point-to-mesh distance function [S1, unofficial].
    d.  **SDF Assignment:**
        $$ \text{sdf} = \begin{cases} -d & \text{if } \text{is\_inside} \\ d & \text{otherwise} \end{cases} $$
        [S1, unofficial]
    e.  **Filtering:**
        *   If `surface_only` mode is active, check for geometric intersection between the voxel cuboid and the mesh. If no intersection, skip [S1, unofficial].
        *   If `narrow_band` is provided, keep the voxel only if $|\text{sdf}| \le \text{narrow\_band}$. Otherwise, keep if `sdf` $\le 0.0$ (inside) [S1, unofficial].
    f.  **Phase Assignment:** If `phase_sphere` is provided, check if the voxel center lies within the sphere ($\text{distance}^2 \le r^2$). Set `phase = 1` if inside, else `phase = 0` [S1, unofficial].
    g.  **Output:** Push a `ParticleData` record with the computed values to the output list [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def slice_based_sdf_voxelization(mesh, bounds_min, bounds_max, resolution, narrow_band=None, phase_sphere=None, surface_only=False):
    # 1. Validation
    if resolution <= 1e-6 or not is_finite(resolution):
        raise ValueError("Invalid resolution")
    if narrow_band is not None and (narrow_band < 0 or not is_finite(narrow_band)):
        raise ValueError("Invalid narrow_band")

    # 2. Grid Setup
    nx = int((bounds_max.x - bounds_min.x) / resolution)
    ny = int((bounds_max.y - bounds_min.y) / resolution)
    nz = int((bounds_max.z - bounds_min.z) / resolution)
    
    output_particles = []

    # 3. Parallel Loop over Y and Z
    for iy in range(ny):
        for iz in range(nz):
            y = bounds_min.y + (iy * resolution) + (resolution * 0.5)
            z = bounds_min.z + (iz * resolution) + (resolution * 0.5)

            # 3a. Ray Casting along X
            start_x = bounds_min.x - 1.0
            ray_origin = Point(start_x, y, z)
            ray_direction = Vector(1, 0, 0)
            max_dist = (max(bounds_max.x, mesh.bounds_max.x) - start_x) + 1.0
            
            hit_xs = []
            current_ray_origin = ray_origin
            while True:
                ray = Ray(current_ray_origin, ray_direction)
                hit_toi = mesh.cast_local_ray(ray, max_dist, true)
                if hit_toi is None:
                    break
                
                hit_point = ray.point_at(hit_toi)
                hit_xs.append(hit_point.x)
                
                # Shift origin slightly to find next intersection
                current_ray_origin = ray.point_at(hit_toi + 1e-4)
                
                if len(hit_xs) > 100_000:
                    break
            
            # 3b. Sort Intersections
            hit_xs.sort()

            # 4. Loop over X
            for ix in range(nx):
                x = bounds_min.x + (ix * resolution) + (resolution * 0.5)
                point = Point(x, y, z)

                # 4a. Inside/Outside Check
                # partition_point returns the index of the first element > x
                # Elements <= x are to the left. Elements > x are to the right.
                # Number of hits to the right = Total hits - Hits <= x
                hits_left_or_at = partition_point(hit_xs, lambda hx: hx <= x)
                intersections_to_right = len(hit_xs) - hits_left_or_at
                is_inside = (intersections_to_right % 2 != 0)

                # 4b. Distance Calculation
                distance = mesh.distance_to_local_point(point, false)
                sdf = -distance if is_inside else distance

                # 4c. Filtering
                keep = False
                if surface_only:
                    # Check if voxel cuboid intersects mesh
                    voxel_iso = Isometry.translation(x, y, z)
                    cuboid = Cuboid(Vector(resolution*0.5, resolution*0.5, resolution*0.5))
                    mesh_iso = Isometry.identity()
                    if intersection_test(mesh_iso, mesh, voxel_iso, cuboid):
                        keep = True
                else:
                    if narrow_band is not None:
                        keep = abs(sdf) <= narrow_band
                    else:
                        keep = sdf <= 0.0

                if not keep:
                    continue

                # 4d. Phase Assignment
                phase = 0
                if phase_sphere is not None:
                    dx = x - phase_sphere[0]
                    dy = y - phase_sphere[1]
                    dz = z - phase_sphere[2]
                    r = phase_sphere[3]
                    if (dx*dx + dy*dy + dz*dz) <= (r*r):
                        phase = 1

                # 4e. Output
                output_particles.append(ParticleData(
                    x=x, y=y, z=z, sdf=sdf, phase=phase,
                    label_id=0, fiber_x=0.0, fiber_y=0.0
                ))

    return output_particles
```

## Operating characteristics
*   **Time Complexity:** $O(N_y N_z \cdot (K \log K + N_x \log K))$, where $N_y, N_z$ are grid dimensions, $K$ is the number of triangle intersections per slice, and $N_x$ is the number of voxels per slice. Sorting `hit_xs` takes $O(K \log K)$, and binary search for each voxel takes $O(\log K)$ [S1, unofficial].
*   **Memory Complexity:** $O(N_x N_y N_z)$ for the output particles, plus $O(K)$ per slice for intersection storage [S1, unofficial].
*   **Performance:** Parallelization over Y and Z slices ensures fine-grained workload distribution. Iterating X sequentially within a slice optimizes spatial cache locality for BVH traversal [S1, unofficial].
*   **Strengths:** Robust inside/outside determination via parity; exact SDF values; supports narrow bands and phase tracking [S1, unofficial].
*   **Limitations:** Performance degrades with high triangle counts per slice ($K$); limited by the 100,000 hit cap per slice [S1, unofficial].
*   **When to use:** When exact SDF values and robust topological containment are required for scanned surfaces [S1, unofficial].
*   **When not to use:** For very large meshes with high intersection density per slice, or when approximate SDF is sufficient and faster hierarchical methods are preferred [S1, unofficial].

## Background
This method is implemented in the `voxelizer-rs` library, an unofficial Rust-based voxelization tool [S1, unofficial]. It is designed for processing triangulated surface meshes (e.g., `.stl` files) into signed distance fields for volume rendering or simulation [S1, unofficial].

## Sources
[S1] voxelizer-rs, https://github.com/lhemerly/voxelizer-rs, fetched 2026-10-28 (unofficial)
