# Hardware Ray Tracing Voxel Occlusion

Also called: Hardware Ray Tracing for Occlusion, Hardware ray-tracing occlusion path, Hardware-RT Voxel Traversal (RT Variant), Hardware-RT voxel path, Hardware-RT primary trace, RT occlusion, GPU RT rays

## Overview
This method determines if a ray from `origin` to `dir` is occluded by any solid voxel within `max_dist` by using hardware ray-tracing cores to traverse a world BVH (TLAS/BLAS) to find candidate brick AABBs, then using a shared software Amanatides-Woo DDA within each candidate brick to resolve the exact first solid voxel [S1, unofficial]. It replaces a software beam pre-pass and hierarchical-DDA empty-space skip with hardware traversal for the coarse search [S1, unofficial]. It differs from the software variant by using hardware RT cores for coarse BVH traversal instead of a software beam pre-pass and hierarchical-DDA, while the in-brick resolution (Amanatides-Woo DDA) is shared between both variants [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origin `origin` (world space), ray direction `dir` (normalized), maximum traversal distance `max_dist`, and the hardware acceleration structures (BLAS/TLAS) representing the voxel world.
*   **Outputs:** Boolean `true` if occluded, `false` otherwise. In full intersection mode, it returns the nearest hit voxel index, material, and normal.

**Data Structures:**
The method relies on specific memory layouts to ensure compatibility with hardware BLAS build requirements [S1, unofficial].

1.  **Brick Structure:** Represents the occupancy and material data for a single brick (4x4x4 voxels).
    ```wgsl
    struct Brick {
        occ_lo: u32,
        occ_hi: u32,
        materials: array<u32, 16>,
    };
    ```
    *   `occ_lo` and `occ_hi`: Two 32-bit integers packing 64 bits total for the 4^3=64 voxels.
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
    *   **Brick Map:** `array<u32>` mapping primitive indices from the BLAS to brick indices in the `bricks` array.

**Parameters:**
*   `RT_PRIMARY`: An `f32` override parameter. Default is 0.0 (off). If > 0.5, enables hardware RT primary trace instead of software DDA [S1, unofficial].
*   `VOXELG_RT`: Environment variable. Setting it to `1` enables hardware RT on a ray-query-capable adapter [S1, unofficial].

**Algorithm Steps:**
The following pseudocode implements the occlusion check. It uses the Vulkan Ray Query API semantics adapted for WebGPU/WGSL style ray queries.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
MAX_DIST_CAP = 1.0e9
NEAREST_HIT_INIT = 1.0e30

def hardware_rt_voxel_occlusion(origin, dir, max_dist, acc_structure, bricks, brick_map, aabbs):
    """
    Determines if a ray is occluded by any solid voxel using hardware RT.
    
    Args:
        origin: vec3<f32> World space origin.
        dir: vec3<f32> Normalized direction.
        max_dist: f32 Maximum distance to search.
        acc_structure: Acceleration structure handle (TLAS/BLAS).
        bricks: Array<Brick> Voxel data.
        brick_map: Array<u32> Maps primitive index to brick index.
        aabbs: Array<GpuAabb> Brick bounding boxes.
        
    Returns:
        bool: True if occluded, False otherwise.
    """
    
    # 1. Rebase origin to window-local coordinates if necessary.
    # Note: The source notes mention rebasing origin to window-local:
    # o = origin - camera.world_origin. 
    # However, the ray query initialization in the main compute shader 
    # uses 'o' directly as the ray origin. We assume 'origin' passed here
    # is already in the coordinate space expected by the acceleration structure.
    # If window-local rebasing is required by the specific implementation context:
    # o = origin - camera.world_origin
    o = origin 

    # 2. Initialize hardware ray query.
    # RayDesc flags: 0u (no special flags), 0xFFu (all hit types).
    # Distance range: [0.0, max_dist].
    rq = rayQueryInitialize(acc_structure, RayDesc(0u, 0xFFu, 0.0, max_dist, o, dir))

    # 3. Track the nearest resolved voxel.
    best_t = NEAREST_HIT_INIT
    is_occluded = False

    # 4. Loop while the ray query proceeds.
    while rayQueryProceed(rq):
        
        # 5. Get candidate intersection.
        c = rayQueryGetCandidateIntersection(rq)
        
        # 6. Check if candidate is an AABB (brick).
        if c.kind == RAY_QUERY_INTERSECTION_AABB:
            
            # 7. Retrieve brick index and AABB.
            primitive_idx = c.primitive_index
            bi = i32(brick_map[primitive_idx])
            a = aabbs[primitive_idx]
            
            # 8. Check if brick is active.
            # This check ensures correctness during async BVH rebuilds.
            # Stale primitives render as sky, matching software DDA behavior.
            if not rt_brick_active(bi):
                continue
            
            # 9. Extract AABB min corner.
            bmin = vec3<f32>(a.min_x, a.min_y, a.min_z)
            
            # 10. Resolve exact voxel hit using in-brick DDA.
            # This calls the shared software Amanatides-Woo DDA.
            # It returns a hit structure 'bh' or indicates no hit.
            bh = resolve_brick_full(bi, bmin, o, dir, origin, best_t, skip_transparent=False)
            
            # 11. Check if a hit was found and if it is closer than current best.
            if bh.hit and bh.t >= 0.0 and bh.t < best_t:
                best_t = bh.t
                is_occluded = True
                
                # 12. Cull further bricks by generating an intersection.
                # This tells the hardware RT core to stop searching beyond this distance.
                rayQueryGenerateIntersection(rq, bh.t)
                
                # Optimization: If we only care about binary occlusion,
                # we can return immediately.
                return True

    # 13. If loop completes without finding a closer hit, return result.
    return is_occluded

def resolve_brick_full(bi, bmin, o, dir, origin, current_best_t, skip_transparent):
    """
    Performs the in-brick Amanatides-Woo DDA to find the first solid voxel.
    This function is shared with the software variant.
    """
    # Implementation details of the in-brick DDA are defined in 
    # [Ray-Cube Intersection (Brick/Slab)](3620-ray-cube-intersection-brick-slab.md).
    # It returns a Hit structure with t, hit flag, voxel coords, material, and normal.
    
    # Pseudocode for the DDA logic (derived from parent article context):
    # 1. Calculate entry/exit times for the brick AABB.
    # 2. Step through voxels using DDA.
    # 3. Check occupancy mask (occ_lo/occ_hi) for each voxel.
    # 4. If solid, return hit info.
    # 5. If no solid voxel found in brick, return hit=False.
    
    # Note: The exact DDA implementation is not repeated here to avoid duplication,
    # but it must be implemented as described in the parent article.
    # For the purpose of this article's completeness, we assume this function
    # correctly implements the Amanatides-Woo traversal within the 4x4x4 brick.
    
    hit_result = perform_in_brick_dda(bi, bmin, o, dir)
    return hit_result

def perform_in_brick_dda(bi, bmin, o, dir):
    """
    Placeholder for the specific Amanatides-Woo DDA implementation 
    found in shaders/rt_voxel_query.wgsl.
    """
    # This function traverses the 4^3 voxels of the brick.
    # It uses the occupancy bits from bricks[bi].occ_lo and bricks[bi].occ_hi.
    # It returns a Hit struct.
    pass
```

**Edge Cases and Pitfalls:**
*   **AABB Packing:** Using `vec3<f32>` for AABB min/max in WGSL causes 16-byte alignment, breaking the tight 24-byte packing required by the BLAS build. Scalar fields must be used [S1, unofficial].
*   **BVH Staleness:** The BVH may be stale during async rebuilds. Correctness is maintained because the tile occupancy mask (`rt_brick_active`) is updated synchronously. Stale primitives render as sky, matching software DDA behavior [S1, unofficial].
*   **Precision:** The committed intersection from the RT core is used only as a cross-check, not the source of truth for the voxel hit. The exact voxel hit is derived from the software DDA to ensure precision [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Coarse traversal is $O(\log N)$ where $N$ is the number of bricks, handled by hardware. Fine traversal is $O(1)$ per candidate brick (fixed 4x4x4 size).
*   **Memory Complexity:** Requires storage for BLAS/TLAS, brick occupancy data, and AABBs. The tight packing of AABBs (24 bytes) minimizes BLAS memory footprint.
*   **Performance:** Offloads coarse empty-space skipping to hardware RT cores, potentially offering significant speedups over software hierarchical DDA for primary rays and shadows [S1, unofficial].
*   **Strengths:** Hardware-accelerated BVH traversal; consistent results with software path via shared DDA; handles async rebuilds gracefully.
*   **Limitations:** Requires ray-query-capable hardware; overhead of software DDA per candidate brick; strict memory layout requirements for BLAS.
*   **When to Use:** When hardware RT cores are available and primary ray/shadow performance is critical. Enabled via `RT_PRIMARY > 0.5` or `VOXELG_RT=1`.
*   **When Not to Use:** On hardware without ray-tracing support; when software fallback is required for correctness in all scenarios.

## Background
This method is part of the voxelG rendering engine, an unofficial project by mstampfli [S1, unofficial]. It leverages modern GPU ray-tracing capabilities (specifically Vulkan Ray Query or WebGPU Ray Query) to optimize voxel traversal. The technique was developed to replace the software beam pre-pass and hierarchical-DDA empty-space skip with hardware traversal, improving performance for primary rays and occlusion queries [S1, unofficial]. The in-brick DDA logic is shared with the software variant to ensure identical rendering results [S1, unofficial].

## Sources
[S1] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
