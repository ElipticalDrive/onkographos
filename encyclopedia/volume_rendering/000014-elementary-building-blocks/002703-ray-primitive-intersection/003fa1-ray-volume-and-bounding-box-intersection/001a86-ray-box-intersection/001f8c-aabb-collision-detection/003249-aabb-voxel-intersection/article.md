# AABB Voxel Intersection
Also called: Brute-force voxel AABB test, Voxel Grid Collision Check, Voxel Sprite AABB Intersection, getCollisions

## Overview
AABB Voxel Intersection determines whether an Axis-Aligned Bounding Box (AABB) overlaps with solid voxels in a voxel world or between two voxel sprites [S1, S3, unofficial]. It solves spatial collision detection in discrete grid environments by iterating through every integer coordinate covered by the query box [S3, unofficial]. Unlike hierarchical traversal methods, it performs a direct brute-force check of all voxel coordinates within the bounding volume [S1, S3, unofficial].

## How it works
**Inputs and Outputs:**
- Input: An AABB defined by floating-point bounds $(min_x, min_y, min_z)$ and $(max_x, max_y, max_z)$, and a voxel data structure providing a coordinate lookup function [S1, S3, unofficial].
- Output: A boolean indicating intersection, or a mapping of collision faces to voxel positions [S2, S3, unofficial].

**Data Structures:**
- The voxel world or sprites store voxels accessible via coordinate lookup functions such as `is_solid(x, y, z)` or `getVoxel(x, y, z)` [S1, S3, unofficial].
- The AABB is defined by floating-point minimum and maximum coordinates [S3, unofficial].
- Collision response data maps face directions to arrays of collision positions [S2, unofficial].

**Numbered Steps:**
1. Compute integer bounds for the query AABB.
2. Iterate through all integer coordinates $(x, y, z)$ within the computed bounds [S1, S3, unofficial].
3. Query the voxel data structure at each coordinate [S3, unofficial].
4. Return true immediately upon finding a collision; return false if all coordinates are checked without a match [S1, S3, unofficial].
5. For sprite-to-sprite checks, compute local offsets for both sprites and verify both voxels are non-null [S1, unofficial].
6. For detailed collision reporting, populate a face-direction map with collision positions [S2, unofficial].

**Formulas:**
- $x_{min} = \lfloor min_x + \epsilon \rfloor$
- $x_{max} = \lfloor max_x - \epsilon \rfloor$
- $y_{min} = \lfloor min_y + \epsilon \rfloor$
- $y_{max} = \lfloor max_y - \epsilon \rfloor$
- $z_{min} = \lfloor min_z + \epsilon \rfloor$
- $z_{max} = \lfloor max_z - \epsilon \rfloor$
- Where $min_*$ and $max_*$ are the AABB floating-point bounds, $\epsilon$ is the safety margin, and $\lfloor \cdot \rfloor$ denotes the floor function [S3, unofficial].
- Local offset for sprite collision: $(x - pos_1.x, y - pos_1.y, z - pos_1.z)$ and $(x - pos_2.x, y - pos_2.y, z - pos_2.z)$ [S1, unofficial].

**Parameters:**
- $\epsilon = 0.01f$: A floating-point safety margin added to minimum bounds and subtracted from maximum bounds to avoid precision issues when flooring coordinates [S3, unofficial].

**Edge Cases and Pitfalls:**
- Precision: Floating-point coordinates near integer boundaries can cause voxels to be missed or double-counted if flooring is not handled with an epsilon margin [S3, unofficial].
- Empty AABB: If the AABB has zero or negative volume, the iteration range becomes empty and the function returns false [S3, unofficial].
- Large AABB: Iterating a very large volume requires many coordinate checks and degrades performance [S1, unofficial].
- Loop bounds discrepancy: Source [S1, unofficial] iterates with an exclusive maximum bound ($z < maxZ$), while Source [S3, unofficial] iterates with an inclusive maximum bound ($x \leq \lfloor max_x - \epsilon \rfloor$) [S1, S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def aabb_voxel_intersection(aabb_min, aabb_max, world, eps=0.01):
    """
    Checks if an AABB intersects any solid voxel in the world.
    
    Args:
        aabb_min: Tuple (min_x, min_y, min_z)
        aabb_max: Tuple (max_x, max_y, max_z)
        world: Object with method is_solid(x, y, z) -> bool
        eps: Floating-point safety margin (default 0.01)
        
    Returns:
        bool: True if collision detected, False otherwise
    """
    x_min = math.floor(aabb_min[0] + eps)
    x_max = math.floor(aabb_max[0] - eps)
    y_min = math.floor(aabb_min[1] + eps)
    y_max = math.floor(aabb_max[1] - eps)
    z_min = math.floor(aabb_min[2] + eps)
    z_max = math.floor(aabb_max[2] - eps)
    
    for z in range(z_min, z_max + 1):
        for y in range(y_min, y_max + 1):
            for x in range(x_min, x_max + 1):
                if world.is_solid(x, y, z):
                    return True
    return False
```

## Operating characteristics
- Time complexity is proportional to the number of voxel coordinates intersected by the AABB [S1, S3, unofficial].
- Memory complexity is $O(1)$ auxiliary space for the boolean check [S1, S3, unofficial].
- Strengths include simplicity and exact results without preprocessing [S1, S3, unofficial].
- Limitations include performance degradation as AABB volume increases [S1, unofficial].

## Background
AABB Voxel Intersection is a fundamental technique in voxel-based game engines and simulations for collision detection between entities and the world, or between voxel sprites [S1, S2, S3, unofficial]. It is commonly implemented in C++ and JavaScript for game development [S1, S2, S3, unofficial].

## Sources
[S1] JamesRandall/SimpleVoxelEngine, https://github.com/JamesRandall/SimpleVoxelEngine, fetched 2026-10-26 (unofficial)
[S2] voxel/voxel-engine-stackgl, https://github.com/voxel/voxel-engine-stackgl, fetched 2026-10-27 (unofficial)
[S3] Linirby/HelloVoxel, https://github.com/Linirby/HelloVoxel, fetched 2026-10-27 (unofficial)
