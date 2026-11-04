# Voxel Raycasting for Block Interaction

Also called: voxel raycast

## Overview
A brute-force raycasting algorithm used to determine block intersection for player interaction, such as breaking or placing blocks in a voxel world [S1, unofficial]. It differs from optimized grid traversal methods (like DDA) by stepping along the ray in fixed increments, offering simplicity at the cost of potential precision gaps [S1, unofficial].

## How it works
**Inputs:**
- `origin`: The starting 3D position of the ray [S1, unofficial].
- `dir`: The normalized direction vector of the ray [S1, unofficial].
- `maxDist`: The maximum distance to search for a solid block [S1, unofficial].
- `solidAt(x, y, z)`: A function returning true if the voxel at integer coordinates `(x, y, z)` is solid [S1, unofficial].

**Outputs:**
- `hitPos`: The integer coordinates of the first solid voxel hit [S1, unofficial].
- `hitNormal`: The surface normal of the hit voxel face [S1, unofficial].
- `hit`: Boolean indicating if a solid block was found [S1, unofficial].

**Data Structures:**
- `pos`: Current 3D float position along the ray [S1, unofficial].
- `prev`: Previous 3D float position along the ray [S1, unofficial].
- `diff`: Difference vector between `pos` and `prev` [S1, unofficial].

**Parameters:**
- `t_step`: Fixed increment for the ray parameter `t`. Typical value: `0.05f` [S1, unofficial].
- `maxDist`: Maximum ray length. Typical value: `6.0f` [S1, unofficial].

**Steps:**
1. Initialize `prev = origin` and `pos = origin` [S1, unofficial].
2. Loop for `t` from `0` to `maxDist` with increment `t_step`:
   a. Calculate current position: `pos = origin + dir * t` [S1, unofficial].
   b. Convert position to integer voxel coordinates:
      $$x = \lfloor pos.x \rfloor$$
      $$y = \lfloor pos.y \rfloor$$
      $$z = \lfloor pos.z \rfloor$$ [S1, unofficial].
   c. Check if voxel at `(x, y, z)` is solid using `solidAt(x, y, z)` [S1, unofficial].
   d. If solid:
      i. Set `hitPos = {x, y, z}` [S1, unofficial].
      ii. Calculate difference vector: `diff = pos - prev` [S1, unofficial].
      iii. Determine hit normal based on the largest absolute component of `diff`:
           - If $|diff.x|$ is largest:
             - If $diff.x > 0$, normal is $\{-1, 0, 0\}$ [S1, unofficial].
             - Else, normal is $\{1, 0, 0\}$ [S1, unofficial].
           - Else if $|diff.y|$ is largest:
             - If $diff.y > 0$, normal is $\{0, -1, 0\}$ [S1, unofficial].
             - Else, normal is $\{0, 1, 0\}$ [S1, unofficial].
           - Else ($|diff.z|$ is largest):
             - If $diff.z > 0$, normal is $\{0, 0, -1\}$ [S1, unofficial].
             - Else, normal is $\{0, 0, 1\}$ [S1, unofficial].
      iv. Set `hit = true` and break loop [S1, unofficial].
   e. Update `prev = pos` [S1, unofficial].
3. If loop completes without finding a solid block, set `hit = false` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def voxel_raycast(origin, dir, max_dist, t_step, solid_at):
    prev = origin
    t = 0.0
    while t <= max_dist:
        pos = origin + dir * t
        x = math.floor(pos.x)
        y = math.floor(pos.y)
        z = math.floor(pos.z)
        
        if solid_at(x, y, z):
            hit_pos = (x, y, z)
            diff = pos - prev
            
            abs_diff_x = abs(diff.x)
            abs_diff_y = abs(diff.y)
            abs_diff_z = abs(diff.z)
            
            if abs_diff_x >= abs_diff_y and abs_diff_x >= abs_diff_z:
                if diff.x > 0:
                    normal = (-1, 0, 0)
                else:
                    normal = (1, 0, 0)
            elif abs_diff_y >= abs_diff_x and abs_diff_y >= abs_diff_z:
                if diff.y > 0:
                    normal = (0, -1, 0)
                else:
                    normal = (0, 1, 0)
            else:
                if diff.z > 0:
                    normal = (0, 0, -1)
                else:
                    normal = (0, 0, 1)
            
            return True, hit_pos, normal
            
        prev = pos
        t += t_step
        
    return False, None, None
```

**Edge Cases and Pitfalls:**
- **Precision/Gap Issue:** The fixed step size (`0.05f`) may skip over thin blocks or gaps smaller than the step if the ray direction is aligned with axes [S1, unofficial]. It is robust for general diagonal movement in a voxel grid but may fail for very thin structures [S1, unofficial].

## Subtopics

### [Voxel Block Interaction (Raycast Hit Detection)](3912-voxel-block-interaction-raycast-hit-detection.md)
A method to find the solid voxel block closest to a raycast hit location by checking the hit block and its 6 immediate neighbors. It differs from brute-force stepping by limiting the search to a constant number of grid cells [G].

## Operating characteristics
- **Time Complexity:** $O(maxDist / t\_step)$ [G]. The loop runs a fixed number of times based on the maximum distance and step size [G].
- **Memory Complexity:** $O(1)$ [G]. Only a few variables are stored [G].
- **Strengths:** Simple to implement; no complex grid traversal logic required [G].
- **Limitations:** May miss thin blocks due to fixed step size; performance degrades linearly with `maxDist` and inversely with `t_step` [S1, unofficial].
- **When to use:** Simple voxel games where block sizes are large relative to the step size and performance requirements are low [G].
- **When not to use:** When high precision is required for thin structures or when `maxDist` is very large [G].

## Background
Originating from early voxel game implementations, this method is a straightforward approach to block interaction [S1, unofficial]. It is documented in the `whudsontosh/CPU-Rasterizer` repository [S1, unofficial].

## Sources
[S1] whudsontosh/CPU-Rasterizer, https://github.com/whudsontosh/CPU-Rasterizer, fetched 2026-10-27 (unofficial)
