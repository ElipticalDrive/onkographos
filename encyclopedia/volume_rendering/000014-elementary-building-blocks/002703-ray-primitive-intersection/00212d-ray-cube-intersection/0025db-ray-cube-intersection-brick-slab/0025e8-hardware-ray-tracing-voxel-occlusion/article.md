# Hardware Ray Tracing Voxel Occlusion

Also called: Hardware Ray Tracing for Occlusion, Hardware ray-tracing occlusion path, Hardware-RT Voxel Traversal (RT Variant), Hardware-RT voxel path, Hardware-RT primary trace, RT occlusion, GPU RT rays

## Overview
This method determines if a ray from `origin` to `dir` is occluded by any solid voxel within `max_dist` by using hardware ray-tracing cores to traverse a world BVH (TLAS/BLAS) to find candidate brick AABBs, then using a shared software Amanatides-Woo DDA within each candidate brick to resolve the exact first solid voxel [S1, unofficial]. It replaces a software beam pre-pass and hierarchical-DDA empty-space skip with hardware traversal for the coarse search [S1, unofficial]. It differs from the software variant by using hardware RT cores for coarse BVH traversal instead of a software beam pre-pass and hierarchical-DDA, while the in-brick resolution (Amanatides-Woo DDA) is shared between both variants [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origin `origin` (world space), ray direction `dir` (normalized), maximum traversal distance `max_dist`, and the hardware acceleration structures (BLAS/TLAS) representing the voxel world [S1, unofficial].
*   **Outputs:** Boolean `true` if occluded, `false` otherwise. In full intersection mode, it returns the nearest hit voxel index, material, and normal [S1, unofficial].

**Data Structures:**
The method relies on specific memory layouts to ensure compatibility with hardware BLAS build requirements [S1, unofficial].

1.  **Brick Structure:** Represents the occupancy and material data for a single brick (4x4x4 voxels) [S1, unofficial].
    ```wgsl
    struct Brick {
        occ_lo: u32,
        occ_hi: u32,
        materials: array<u32, 16>,
    };
    ```
    *   `occ_lo` and `occ_hi`: Two 32-bit integers packing 64 bits total for the 4^3=64 voxels [S1, unofficial].
    *   `materials`: Array of 16 `u32` values. Materials are packed 4 per 32-bit word (8 bits per material) [S1, unofficial].

2.  **AABB Structure:** Represents the bounding box of a brick. Scalar fields are used to maintain tight 24-byte packing, avoiding WGSL `vec3` 16-byte alignment padding which would break BLAS build compatibility [S1, unofficial].
    ```wgsl
    struct GpuAabb {
        min_x: f32, min_y: f32, min_z: f32,
        max_x: f32, max_y: f32, max_z: f32,
        _p0: f32, _p1: f32,
    };
    ```
    *   Total size: 36 bytes (9 floats) [S1, unofficial].

3.  **Acceleration Structures:**
    *   **BLAS:** Contains one `GpuAabb` per non-empty brick. The RT core skips empty space in hardware [S1, unofficial].
    *   **TLAS:** World Top-Level Acceleration Structure containing references to the BLAS.
    *   **Brick Map:** `array<u32>` mapping primitive indices from the BLAS to brick indices in the `bricks` array [S1, unofficial].

**Parameters:**
*   `RT_PRIMARY`: An `f32` override parameter. Default is 0.0 (off). If > 0.5, enables hardware RT primary trace instead of software DDA [S1, unofficial].
*   `VOXELG_RT`: Environment variable. Setting it to `1` enables hardware RT on a ray-query-capable adapter [S1, unofficial].

**Algorithm Steps:**
The method initializes a hardware ray query, iterates through candidate brick AABBs, checks brick activity, resolves the exact voxel hit via a shared DDA, and culls further traversal upon finding a closer hit [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hardware_rt_voxel_occlusion(origin, dir, max_dist, tlas, bricks, brick_map, aabbs, camera_world_origin):
    # 1. Rebase world-space origin to window-local coordinates
    o = origin - camera_world_origin

    # 2. Initialize hardware ray query against TLAS
    # RayDesc flags: 0 (no special flags), 0xFF (all hit types)
    # Distance range: [0.0, max_dist]
    rq = ray_query_init(tlas, origin=o, direction=dir, t_min=0.0, t_max=max_dist, flags=0, hit_mask=0xFF)

    best_t = 1e30
    occluded = False

    # 3. Loop while the ray query proceeds
    while ray_query_proceed(rq):
        # 4. Get candidate intersection
        c = ray_query_get_candidate(rq)

        # 5. Check if candidate is an AABB (brick)
        if c.kind == RAY_QUERY_INTERSECTION_AABB:
            # 6. Retrieve brick index and AABB
            primitive_idx = c.primitive_index
            bi = int(brick_map[primitive_idx])
            a = aabbs[primitive_idx]

            # 7. Check if brick is active
            if not rt_brick_active(bi):
                continue

            # 8. Extract AABB min corner
            bmin = (a.min_x, a.min_y, a.min_z)

            # 9. Resolve exact voxel hit using in-brick DDA
            # Calls shared Amanatides-Woo DDA from parent article
            bh = resolve_brick_full(bi, bmin, o, dir, origin, best_t, skip_transparent=False)

            # 10. Check if hit found and closer than current best
            if bh.hit and bh.t >= 0.0 and bh.t < best_t:
                best_t = bh.t
                occluded = True

                # 11. Cull further bricks by generating an intersection
                ray_query_generate_intersection(rq, bh.t)

                # Optimization: Return immediately for binary occlusion check
                return True

    # 12. If loop completes without finding a closer hit
    return occluded
```

**Edge Cases and Pitfalls:**
*   **AABB Packing:** Using `vec3<f32>` for AABB min/max in WGSL causes 16-byte alignment, breaking the tight 24-byte packing required by the BLAS build. Scalar fields are used instead [S1, unofficial].
*   **BVH Staleness:** The BVH may be stale during async rebuilds. Correctness is maintained because the tile occupancy mask (`rt_brick_active`) is updated synchronously. Stale primitives render as sky, matching software DDA behavior [S1, unofficial].
*   **Precision:** The committed intersection from the RT core is used only as a cross-check, not the source of truth for the voxel hit. The exact voxel hit is derived from the software DDA to ensure precision [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Offloads coarse empty-space skipping to hardware RT cores, potentially offering significant speedups over software hierarchical DDA for primary rays and shadows [S1, unofficial].
*   **Strengths:** Hardware-accelerated BVH traversal; consistent results with software path via shared DDA; handles async rebuilds gracefully [S1, unofficial].
*   **Limitations:** Requires ray-query-capable hardware; overhead of software DDA per candidate brick; strict memory layout requirements for BLAS [S1, unofficial].
*   **When to Use:** When hardware RT cores are available and primary ray/shadow performance is critical. Enabled via `RT_PRIMARY > 0.5` or `VOXELG_RT=1` [S1, unofficial].
*   **When Not to Use:** On hardware without ray-tracing support; when software fallback is required for correctness in all scenarios. Not given in the sources.

## Background
This method originates from the voxelG project [S1, unofficial]. It leverages modern GPU ray-tracing capabilities (specifically Vulkan Ray Query or WebGPU Ray Query) to optimize voxel traversal. The technique was developed to replace the software beam pre-pass and hierarchical-DDA empty-space skip with hardware traversal, improving performance for primary rays and occlusion queries [S1, unofficial]. The in-brick DDA logic is shared with the software variant to ensure identical rendering results [S1, unofficial].

## Sources
[S1] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
