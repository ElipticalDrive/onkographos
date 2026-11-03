# In-Brick DDA for Occlusion
## Overview
A Digital Differential Analyzer (DDA) that marches through a 4x4x4 voxel brick to determine if any solid voxel occludes a shadow ray [S1, unofficial]. Unlike primary hit resolution which stops at the first intersection, this method visits all solid voxels within the brick to correctly handle transparent or fringe cases [S1, unofficial]. It serves as the core traversal logic for shadow testing within the brick-based ray-cube intersection framework [S1, unofficial].

## How it works
**Inputs:**
- Ray origin `o` and direction `dir` (vec3<f32>).
- Brick bounds `bmin` and `bmax` (vec3<f32>), where `bmax = bmin + vec3<f32>(4.0)`.
- Time bounds `[t_lo, t_hi]` for the ray segment.
- Brick identifier `bi` and voxel data accessors `brick_voxel_idx` and `brick_voxel_solid`.
- Shadow occlusion check function `shadow_voxel_occludes`.

**Outputs:**
- Boolean: `true` if any solid voxel occludes the ray, `false` otherwise.

**Data Structures:**
- The brick is a fixed 4x4x4 grid of voxels.
- Voxel coordinates `v` are integers in `[0, 3]`.
- `brick_voxel_idx(x, y, z)` maps 3D coordinates to a linear index.
- `brick_voxel_solid(bi, vi)` returns true if the voxel at index `vi` in brick `bi` is solid.

**Steps:**
1. Compute inverse direction components, handling zero division safely:
   $$ inv = \text{vec3<f32>}(rt\_safe\_inv(dir.x), rt\_safe\_inv(dir.y), rt\_safe\_inv(dir.z)) $$
2. Compute slab entry/exit times for the brick bounds:
   $$ tb0 = (bmin - o) * inv $$
   $$ tb1 = (bmin + \text{vec3<f32>}(4.0) - o) * inv $$
3. Calculate brick entry `tnear` and exit `tfar` times:
   $$ tnear = \max(\max(\min(tb0.x, tb1.x), \min(tb0.y, tb1.y)), \min(tb0.z, tb1.z)) $$
   $$ tfar = \min(\min(\max(tb0.x, tb1.x), \max(tb0.y, tb1.y)), \max(tb0.z, tb1.z)) $$
4. Clamp start time `t` and end time `t_end`:
   $$ t = \max(tnear, 0.0) $$
   $$ t\_end = \min(tfar, t\_hi) $$
   If $t > t\_end$, return `false`.
5. Calculate initial voxel position and clamp voxel coordinates:
   $$ p = o + dir * (t + 1e-4) - bmin $$
   $$ v = \text{clamp}(\text{vec3<i32>}(floor(p)), \text{vec3<i32>}(0), \text{vec3<i32>}(3)) $$
6. Determine step direction for each axis:
   $$ step.x = \text{select}(-1, 1, dir.x \ge 0.0) $$
   $$ step.y = \text{select}(-1, 1, dir.y \ge 0.0) $$
   $$ step.z = \text{select}(-1, 1, dir.z \ge 0.0) $$
7. Calculate next boundary times `next` and initial `t_max` for each axis, and `t_delta = abs(inv)`.
8. Loop `i` from 0 to 15:
   - Check bounds: if $v.x < 0$ or $v.x > 3$ or $v.y < 0$ or $v.y > 3$ or $v.z < 0$ or $v.z > 3$, return `false`.
   - Check time: if $t\_cur > t\_end$, return `false`.
   - Get voxel index: $vi = brick\_voxel\_idx(v.x, v.y, v.z)$.
   - If $brick\_voxel\_solid(bi, vi)$ is true, check $shadow\_voxel\_occludes(world\_base + v, m, t\_cur, origin\_world, dir)$. If true, return `true`.
   - Step to the next voxel along the axis with the smallest `t_max`. Update `v`, `t_cur`, and `t_max` for that axis. For example, if x-axis is smallest:
     $$ v.x = v.x + step.x $$
     $$ t\_cur = t\_max.x $$
     $$ t\_max.x = t\_max.x + t\_delta.x $$
9. If loop completes without occlusion, return `false`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def in_brick_dda_occlusion(o, dir, bmin, bmax, t_lo, t_hi, brick_voxel_idx, brick_voxel_solid, shadow_voxel_occludes, world_base, m):
    inv = vec3(1.0 / dir.x if dir.x != 0 else float('inf'),
               1.0 / dir.y if dir.y != 0 else float('inf'),
               1.0 / dir.z if dir.z != 0 else float('inf'))
    
    tb0 = (bmin - o) * inv
    tb1 = (bmin + vec3(4.0) - o) * inv
    
    tnear = max(max(min(tb0.x, tb1.x), min(tb0.y, tb1.y)), min(tb0.z, tb1.z))
    tfar = min(min(max(tb0.x, tb1.x), max(tb0.y, tb1.y)), max(tb0.z, tb1.z))
    
    t = max(tnear, 0.0)
    t_end = min(tfar, t_hi)
    
    if t > t_end:
        return False
    
    p = o + dir * (t + 1e-4) - bmin
    v = clamp(floor(p), vec3(0), vec3(3))
    
    step = vec3(1 if dir.x >= 0 else -1,
                1 if dir.y >= 0 else -1,
                1 if dir.z >= 0 else -1)
    
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
    
    for i in range(16):
        if v.x < 0 or v.x > 3 or v.y < 0 or v.y > 3 or v.z < 0 or v.z > 3:
            return False
        if t_cur > t_end:
            return False
            
        vi = brick_voxel_idx(v.x, v.y, v.z)
        if brick_voxel_solid(vi):
            if shadow_voxel_occludes(world_base + v, m, t_cur, o, dir):
                return True
                
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
- Brick size: 4x4x4 voxels.
- Loop limit: 16 iterations.

**Edge Cases and Pitfalls:**
- Zero direction components: handled by `rt_safe_inv` which returns infinity.
- Ray not intersecting brick: handled by `t > t_end` check.
- Voxel coordinates out of bounds: checked at the start of each loop iteration.
- Precision issues: small epsilon `1e-4` added to ray origin to avoid self-intersection.

## Operating characteristics
- Time Complexity: O(1) per brick, as the loop runs a fixed 16 times.
- Memory Complexity: O(1) additional memory.
- Strengths: Efficient for shadow testing within small bricks, handles transparency by checking all solid voxels.
- Limitations: Fixed brick size limits flexibility, may be less efficient for very sparse or very dense bricks compared to other methods.
- When to use: Shadow testing in voxel-based rendering with small brick sizes.
- When not to use: Large bricks or when primary hit detection is sufficient.

## Background
Originated in the voxelG project by mstampfli [S1, unofficial]. It is part of the elementary building blocks for ray-primitive intersection in volume rendering.

## Sources
[S1] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
