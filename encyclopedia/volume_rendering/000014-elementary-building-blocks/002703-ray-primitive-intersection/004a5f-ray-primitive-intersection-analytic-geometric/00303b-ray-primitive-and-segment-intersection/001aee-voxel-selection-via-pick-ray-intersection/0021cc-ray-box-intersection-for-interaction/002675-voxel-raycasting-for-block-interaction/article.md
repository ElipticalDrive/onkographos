# Voxel Raycasting for Block Interaction

Also called: Voxel Block Interaction via Raycasting, voxel raycast

## Overview
A brute-force raycasting algorithm used to determine block intersection for player interaction, such as breaking or placing blocks in a voxel world [S1, unofficial]. It steps along a ray in fixed increments to find the first solid voxel [S1, unofficial]. It differs from its parent, [Ray-Box Intersection for Interaction](2638-ray-box-intersection-for-interaction.md), which uses analytical intersection tests, by using discrete sampling; and from its child, [Voxel Block Interaction (Raycast Hit Detection)](3912-voxel-block-interaction-raycast-hit-detection.md), which performs a localized search around a known hit location rather than stepping from the origin [G].

## How it works
**Inputs:**
- `origin`: The starting position of the ray (Vec3) [S1, unofficial].
- `dir`: The normalized direction of the ray (Vec3) [S1, unofficial].
- `maxDist`: The maximum distance to search (float) [S1, unofficial].
- `t_step`: The fixed increment for the ray step (float) [S1, unofficial].
- `solidAt(x, y, z)`: A function returning true if the voxel at integer coordinates `(x, y, z)` is solid [S1, unofficial].

**Outputs:**
- `hitPos`: The integer coordinates of the first solid voxel found (Vec3) [S1, unofficial].
- `normal`: The face normal of the hit voxel (Vec3) [S1, unofficial].
- Boolean indicating success [S1, unofficial].

**Data Structures:**
- `pos`: Current ray position (Vec3) [S1, unofficial].
- `prev`: Previous ray position (Vec3) [S1, unofficial].
- `diff`: Difference vector between current and previous positions (Vec3) [S1, unofficial].

**Steps:**
1. Initialize `prev` to `origin` [S1, unofficial].
2. Loop for `t` from `0` to `maxDist` with increment `t_step` [S1, unofficial].
3. Calculate current position: `pos = origin + dir * t` [S1, unofficial].
4. Convert position to integer voxel coordinates:
   $$x = \lfloor pos.x \rfloor$$
   $$y = \lfloor pos.y \rfloor$$
   $$z = \lfloor pos.z \rfloor$$ [S1, unofficial].
5. Check if voxel at `(x, y, z)` is solid using `solidAt(x, y, z)` [S1, unofficial].
6. If solid:
   - Set `hitPos` to `{(float)x, (float)y, (float)z}` [S1, unofficial].
   - Calculate difference vector: `diff = pos - prev` [S1, unofficial].
   - Determine hit normal based on the largest absolute component of `diff`:
     - If $|diff.x|$ is largest:
       - If $diff.x > 0$, normal is `{-1, 0, 0}` [S1, unofficial].
       - Else, normal is `{1, 0, 0}` [S1, unofficial].
     - Else if $|diff.y|$ is largest:
       - If $diff.y > 0$, normal is `{0, -1, 0}` [S1, unofficial].
       - Else, normal is `{0, 1, 0}` [S1, unofficial].
     - Else:
       - If $diff.z > 0$, normal is `{0, 0, -1}` [S1, unofficial].
       - Else, normal is `{0, 0, 1}` [S1, unofficial].
   - Return success [S1, unofficial].
7. Update `prev = pos` [S1, unofficial].
8. If loop completes without finding a solid voxel, return failure [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def voxel_raycast_block_interaction(origin, dir, max_dist, t_step, solid_at):
    prev = origin
    t = 0.0
    while t <= max_dist:
        pos = (origin[0] + dir[0] * t, origin[1] + dir[1] * t, origin[2] + dir[2] * t)
        x = math.floor(pos[0])
        y = math.floor(pos[1])
        z = math.floor(pos[2])
        
        if solid_at(x, y, z):
            hit_pos = (float(x), float(y), float(z))
            diff = (pos[0] - prev[0], pos[1] - prev[1], pos[2] - prev[2])
            
            abs_dx = abs(diff[0])
            abs_dy = abs(diff[1])
            abs_dz = abs(diff[2])
            
            if abs_dx >= abs_dy and abs_dx >= abs_dz:
                normal = (-1.0, 0.0, 0.0) if diff[0] > 0 else (1.0, 0.0, 0.0)
            elif abs_dy >= abs_dx and abs_dy >= abs_dz:
                normal = (0.0, -1.0, 0.0) if diff[1] > 0 else (0.0, 1.0, 0.0)
            else:
                normal = (0.0, 0.0, -1.0) if diff[2] > 0 else (0.0, 0.0, 1.0)
            
            return hit_pos, normal, True
        
        prev = pos
        t += t_step
        
    return None, None, False
```

**Parameters:**
- `maxDist`: Typical value is `6.0f` [S1, unofficial].
- `t_step`: Typical value is `0.05f` [S1, unofficial].

**Edge Cases and Pitfalls:**
- Precision/Gap issue: The fixed step size (`0.05f`) may skip over thin blocks or gaps smaller than the step if the ray direction is aligned with axes, though it is robust for general diagonal movement in a voxel grid [S1, unofficial].

## Subtopics

### [Voxel Block Interaction (Raycast Hit Detection)](3912-voxel-block-interaction-raycast-hit-detection.md)
Identifies the specific solid voxel block intersected by evaluating the hit location's integer grid coordinates and its six immediate neighbors [G]. It solves the problem of discrete block selection in a chunked voxel world where a continuous raycast hit point may lie on the boundary between blocks or within an empty space adjacent to a solid one [G]. It differs from this method by instead performing a localized search around a known hit location to determine the precise block for breaking or placing [G].

## Operating characteristics
**Time Complexity:** $O(\frac{maxDist}{t\_step})$ iterations [G]. With typical values, this is approximately 120 iterations per raycast [G].
**Memory Complexity:** $O(1)$ auxiliary space [G].
**Measured Performance:** Not given in the sources.
**Scaling:** Linear with distance and inverse with step size [G].
**Strengths:** Simple to implement; robust for general diagonal movement in a voxel grid [S1, unofficial].
**Limitations:** May skip thin blocks or gaps smaller than the step size if the ray direction is aligned with axes [S1, unofficial].
**When to use:** Simple voxel worlds where performance is not critical and implementation simplicity is preferred [G].
**When not to use:** When high precision is required for thin structures or when performance is critical for many simultaneous raycasts [G].

## Background
Originated in the context of CPU-based voxel rasterizers and game development, specifically as seen in the `whudsontosh/CPU-Rasterizer` repository [S1, unofficial]. It is used for player interaction in voxel-based games to determine which block is being targeted for breaking or placing [S1, unofficial].

## Sources
[S1] whudsontosh/CPU-Rasterizer, https://github.com/whudsontosh/CPU-Rasterizer, fetched 2026-10-27 (unofficial)
[S2] philip2012/voxel-forge, https://github.com/philip2012/voxel-forge, fetched 2026-10-27 (unofficial)
