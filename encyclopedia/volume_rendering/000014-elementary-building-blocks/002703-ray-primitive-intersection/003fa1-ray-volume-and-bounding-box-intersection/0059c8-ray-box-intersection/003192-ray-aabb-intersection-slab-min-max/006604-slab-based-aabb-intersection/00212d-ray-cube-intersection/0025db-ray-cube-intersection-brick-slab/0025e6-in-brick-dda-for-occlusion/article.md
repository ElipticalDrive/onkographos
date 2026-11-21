# In-Brick DDA for Occlusion
## Overview
A Digital Differential Analyzer (DDA) that marches through a 4×4×4 voxel brick to determine if any solid voxel occludes a shadow ray [S1, unofficial]. Unlike primary hit resolution which stops at the first intersection, this method visits all solid voxels within the brick to correctly handle transparent or fringe cases [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Ray origin `o` and direction `dir` (`vec3<f32>`) [S1, unofficial].
- Brick bounds `bmin` and `bmax` (`vec3<f32>`), where `bmax = bmin + vec3<f32>(4.0)` [S1, unofficial].
- Time bounds `[t_lo, t_hi]` for the ray segment [S1, unofficial].
- Brick identifier `bi` and voxel data accessors `brick_voxel_idx` and `brick_voxel_solid` [S1, unofficial].
- Shadow occlusion check function `shadow_voxel_occludes` [S1, unofficial].
- Output: Boolean `true` if any solid voxel occludes the ray, `false` otherwise [S1, unofficial].

**Data Structures:**
- The brick is a fixed 4×4×4 grid of voxels [S1, unofficial].
- Voxel coordinates `v` are integers in `[0, 3]` [S1, unofficial].
- `brick_voxel_idx(x, y, z)` maps 3D coordinates to a linear index [S1, unofficial].
- `brick_voxel_solid(bi, vi)` returns true if the voxel at index `vi` in brick `bi` is solid [S1, unofficial].

**Numbered Steps and Formulas:**
1. Compute inverse direction components, handling zero division safely: `inv = vec3<f32>(rt_safe_inv(dir.x), rt_safe_inv(dir.y), rt_safe_inv(dir.z))` [S1, unofficial].
2. Compute slab entry/exit times for the brick bounds: `tb0 = (bmin - o) * inv` and `tb1 = (bmin + vec3<f32>(4.0) - o) * inv` [S1, unofficial].
3. Calculate brick entry `tnear` and exit `tfar` times: `tnear = max(max(min(tb0.x, tb1.x), min(tb0.y, tb1.y)), min(tb0.z, tb1.z))` and `tfar = min(min(max(tb0.x, tb1.x), max(tb0.y, tb1.y)), max(tb0.z, tb1.z))` [S1, unofficial]. *(Note: corrected `tb3.z` typo from source to `tb0.z`)*.
4. Clamp start time `t` and end time `t_end`: `t = max(tnear, 0.0)` and `t_end = min(tfar, t_hi)` [S1, unofficial]. If `t > t_end`, return `false` [S1, unofficial].
5. Calculate initial voxel position and clamp voxel coordinates: `p = o + dir * (t + 1e-4) - bmin` and `v = clamp(vec3<i32>(floor(p)), vec3<i32>(0), vec3<i32>(3))` [S1, unofficial].
6. Determine step direction for each axis: `step.x = select(-1, 1, dir.x >= 0.0)`, `step.y = select(-1, 1, dir.y >= 0.0)`, `step.z = select(-1, 1, dir.z >= 0.0)` [S1, unofficial].
7. Calculate next boundary times and initial `t_max` for each axis, and `t_delta = abs(inv)` [S1, unofficial].
8. Loop `i` from 0 to 15:
   - Check bounds: if `v.x < 0` or `v.x > 3` or `v.y < 0` or `v.y > 3` or `v.z < 0` or `v.z > 3`, return `false` [S1, unofficial].
   - Check time: if `t_cur > t_end`, return `false` [S1, unofficial].
   - Get voxel index: `vi = brick_voxel_idx(v.x, v.y, v.z)` [S1, unofficial].
   - If `brick_voxel_solid(bi, vi)` is true, check `shadow_voxel_occludes(world_base + v, m, t_cur, origin_world, dir)` [S1, unofficial]. If true, return `true` [S1, unofficial].
   - Step to the next voxel along the axis with the smallest `t_max` [S1, unofficial]. Update `v`, `t_cur`, and `t_max` for that axis [S1, unofficial].
9. If loop completes without occlusion, return `false` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def in_brick_dda_occlusion(o, dir, bmin, t_lo, t_hi, bi, brick_voxel_idx, brick_voxel_solid, shadow_voxel_occludes, world_base, m):
    # 1. Inverse direction (handles zero components safely)
    inv = vec3(1.0 / dir.x if dir.x != 0 else float('inf'),
               1.0 / dir.y if dir.y != 0 else float('inf'),
               1.0 / dir.z if dir.z != 0 else float('inf'))
    
    # 2. Slab entry/exit times
    tb0 = (bmin - o) * inv
    tb1 = (bmin + vec3(4.0) - o) * inv
    
    # 3. Brick entry/exit times
    tnear = max(max(min(tb0.x, tb1.x), min(tb0.y, tb1.y)), min(tb0.z, tb1.z))
    tfar = min(min(max(tb0.x, tb1.x), max(tb0.y, tb1.y)), max(tb0.z, tb1.z))
    
    # 4. Clamp to ray segment
    t = max(tnear, 0.0)
    t_end = min(tfar, t_hi)
    if t > t_end:
        return False
    
    # 5. Initial voxel position with epsilon offset
    p = o + dir * (t + 1e-4) - bmin
    v = clamp(floor(p), vec3(0), vec3(3))
    
    # 6. Step direction
    step = vec3(1 if dir.x >= 0 else -1,
                1 if dir.y >= 0 else -1,
                1 if dir.z >= 0 else -1)
    
    # 7. Delta and initial t_max (Amanatides-Woo standard initialization)
    t_delta = abs(inv)
    t_max = vec3(0.0, 0.0, 0.0)
    if dir.x >= 0:
        t_max.x = (bmin.x + 1.0 - o.x) * inv.x
    else:
        t_max.x = (bmin.x - o.x) * inv.x
    if dir.y >= 0:
        t_max.y = (bmin.y + 1.0 - o.y) * inv.y
    else:
        t_max.y = (bmin.y - o.y) * inv.y
    if dir.z >= 0:
        t_max.z = (bmin.z + 1.0 - o.z) * inv.z
    else:
        t_max.z = (bmin.z - o.z) * inv.z
        
    t_cur = t
    
    # 8. Fixed 16-iteration loop
    for i in range(16):
        if v.x < 0 or v.x > 3 or v.y < 0 or v.y > 3 or v.z < 0 or v.z > 3:
            return False
        if t_cur > t_end:
            return False
            
        vi = brick_voxel_idx(v.x, v.y, v.z)
        if brick_voxel_solid(bi, vi):
            if shadow_voxel_occludes(world_base + v, m, t_cur, o, dir):
                return True
                
        # Step along axis with smallest t_max
        if t_max.x < t_max.y:
            if t_max.x < t_max.z:
                v.x += step.x
                t_cur = t_max.x
                t_max.x += t_delta.x
            else:
                v.z += step.z
                t_cur = t_max.z
                t_max.z += t_delta.z
        else:
            if t_max.y < t_max.z:
                v.y += step.y
                t_cur = t_max.y
                t_max.y += t_delta.y
            else:
                v.z += step.z
                t_cur = t_max.z
                t_max.z += t_delta.z
                
    return False
```

**Parameters:**
- Brick size: 4×4×4 voxels [S1, unofficial].
- Loop limit: 16 iterations [S1, unofficial].

**Edge Cases and Pitfalls:**
- Zero direction components: handled by `rt_safe_inv` which returns infinity [S1, unofficial].
- Ray not intersecting brick: handled by `t > t_end` check [S1, unofficial].
- Voxel coordinates out of bounds: checked at the start of each loop iteration [S1, unofficial].
- Precision issues: small epsilon `1e-4` added to ray origin to avoid self-intersection [S1, unofficial].

## Operating characteristics
- Time Complexity: O(1) per brick, as the loop runs a fixed 16 times [S1, unofficial].
- Memory Complexity: O(1) additional memory [S1, unofficial].
- Strengths: Efficient for shadow testing within small bricks, handles transparency by checking all solid voxels [S1, unofficial].
- Limitations: Fixed brick size limits flexibility [S1, unofficial].
- When to use: Shadow testing in voxel-based rendering with small brick sizes [S1, unofficial].
- When not to use: Large bricks or when primary hit detection is sufficient [S1, unofficial].

## Background
Developed as part of the elementary building blocks for ray-primitive intersection in volume rendering [S1, unofficial].

## Sources
[S1] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
