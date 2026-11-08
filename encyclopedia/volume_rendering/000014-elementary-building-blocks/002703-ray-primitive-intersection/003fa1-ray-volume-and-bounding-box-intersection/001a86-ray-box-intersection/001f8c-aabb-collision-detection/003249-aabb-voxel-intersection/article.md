# AABB Voxel Intersection
Also called: Brute-force voxel AABB test, Voxel Grid Collision Check, Voxel Sprite AABB Intersection, getCollisions

## Overview
AABB Voxel Intersection determines whether an Axis-Aligned Bounding Box (AABB) overlaps with any solid voxel in a voxel world or between two voxel sprites [S1, S3, unofficial]. It solves the problem of spatial collision detection in discrete grid-based environments by iterating through the integer coordinates covered by the query box [S3, unofficial]. Unlike hierarchical methods such as [Octree Voxel Traversal](52-octree-voxel-traversal.md) or [KD-Tree Ray Casting](52-kd-tree-ray-casting.md), this method performs a direct brute-force check of every voxel coordinate within the bounding volume [S1, S3, unofficial].

## How it works
**Inputs and Outputs:**
- Input: An AABB defined by minimum and maximum coordinates $(min_x, min_y, min_z)$ and $(max_x, max_y, max_z)$, and a voxel world data structure providing a lookup function `is_solid(x, y, z)` or `getVoxel(x, y, z)` [S1, S3, unofficial].
- Output: A boolean indicating intersection, or a mapping of collision faces to voxel positions [S2, S3, unofficial].

**Data Structures:**
- The voxel world is typically stored as a sparse or dense array. For sparse storage, a hash map or list of active voxels is used [S1, unofficial]. For dense storage, a 3D array is used [S3, unofficial].
- The AABB is defined by floating-point bounds [S3, unofficial].

**Parameters:**
- `eps`: A floating-point safety margin, typically $0.01$, added to minimum bounds and subtracted from maximum bounds to avoid precision issues when flooring coordinates [S3, unofficial].

**Steps:**
1. **Compute Intersection Bounds:** Determine the integer range of voxel coordinates to check.
   - Calculate $x_{min} = \lfloor min_x + \epsilon \rfloor$ [S3, unofficial].
   - Calculate $x_{max} = \lfloor max_x - \epsilon \rfloor$ [S3, unofficial].
   - Calculate $y_{min} = \lfloor min_y + \epsilon \rfloor$ [S3, unofficial].
   - Calculate $y_{max} = \lfloor max_y - \epsilon \rfloor$ [S3, unofficial].
   - Calculate $z_{min} = \lfloor min_z + \epsilon \rfloor$ [S3, unofficial].
   - Calculate $z_{max} = \lfloor max_z - \epsilon \rfloor$ [S3, unofficial].
   - If any min > max, return false (no overlap) [S3, unofficial].

2. **Iterate Coordinates:** Loop through all integer coordinates $(x, y, z)$ within the computed bounds [S1, S3, unofficial].
   - Outer loop: $z$ from $z_{min}$ to $z_{max}$ inclusive [S1, S3, unofficial].
   - Middle loop: $y$ from $y_{min}$ to $y_{max}$ inclusive [S1, S3, unofficial].
   - Inner loop: $x$ from $x_{min}$ to $x_{max}$ inclusive [S1, S3, unofficial].

3. **Check Voxel Status:**
   - For each $(x, y, z)$, query the voxel world [S3, unofficial].
   - If using sprite-to-sprite collision, retrieve `thisVoxel` at local offset $(x - pos_1.x, y - pos_1.y, z - pos_1.z)$ and `otherVoxel` at local offset $(x - pos_2.x, y - pos_2.y, z - pos_2.z)$ [S1, unofficial].
   - If `thisVoxel` is not null and `otherVoxel` is not null, a collision is detected [S1, unofficial].
   - If using world collision, if `world.is_solid_at(x, y, z)` returns true, a collision is detected [S3, unofficial].

4. **Return Result:**
   - If a collision is found in step 3, return true immediately [S1, S3, unofficial].
   - If loops complete without finding a collision, return false [S1, S3, unofficial].
   - Alternatively, if detailed collision data is required, return a mapping of face directions (`back`, `bottom`, `down`, `forward`, `left`, `middle`, `right`, `top`, `up`) to arrays of positions where collisions occur [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
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
    # Calculate integer bounds with epsilon safety margin
    x_min = floor(aabb_min[0] + eps)
    x_max = floor(aabb_max[0] - eps)
    y_min = floor(aabb_min[1] + eps)
    y_max = floor(aabb_max[1] - eps)
    z_min = floor(aabb_min[2] + eps)
    z_max = floor(aabb_max[2] - eps)
    
    # Check if bounds are valid
    if x_min > x_max or y_min > y_max or z_min > z_max:
        return False
        
    # Iterate through all voxel coordinates in the AABB
    for z in range(z_min, z_max + 1):
        for y in range(y_min, y_max + 1):
            for x in range(x_min, x_max + 1):
                # Check if the voxel at this coordinate is solid
                if world.is_solid(x, y, z):
                    return True
                    
    return False
```

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point coordinates near integer boundaries can cause voxels to be missed or double-counted if flooring is not handled with an epsilon [S3, unofficial].
- **Empty AABB:** If the AABB has zero or negative volume, the loops should not execute, and the function should return false [S3, unofficial].
- **Large AABBs:** If the AABB is very large, the brute-force iteration may be computationally expensive [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of voxels intersected by the AABB. In the worst case, $N = (x_{max} - x_{min}) \times (y_{max} - y_{min}) \times (z_{max} - z_{min})$ [S1, S3, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space, as it only iterates through coordinates without storing intermediate results (unless returning detailed collision maps) [S1, S3, unofficial].
- **Performance:** Performance is directly proportional to the volume of the AABB and the density of solid voxels [S1, unofficial]. For sparse worlds, hash-based lookups may add overhead [S1, unofficial].
- **Strengths:** Simple to implement; no preprocessing or hierarchical structure required; exact results [S1, S3, unofficial].
- **Limitations:** Inefficient for large AABBs or sparse worlds where most of the volume is empty [S1, unofficial].
- **When to Use:** Small AABBs, dense voxel worlds, or when simplicity is prioritized over performance [S1, S3, unofficial].
- **When Not to Use:** Large AABBs, sparse worlds, or real-time applications requiring high performance with many queries [S1, unofficial].

## Background
AABB Voxel Intersection is a fundamental technique in voxel-based game engines and simulations. It is used in various open-source voxel engines for collision detection between player entities and the world, or between different voxel sprites [S1, S2, S3, unofficial]. The method is often referred to as "brute-force" due to its exhaustive search of the bounding volume [S1, unofficial]. It is commonly implemented in C++, JavaScript, and other languages for game development [S1, S2, S3, unofficial].

## Sources
[S1] JamesRandall/SimpleVoxelEngine, https://github.com/JamesRandall/SimpleVoxelEngine, fetched 2026-10-26 (unofficial)
[S2] voxel/voxel-engine-stackgl, https://github.com/voxel/voxel-engine-stackgl, fetched 2026-10-27 (unofficial)
[S3] Linirby/HelloVoxel, https://github.com/Linirby/HelloVoxel, fetched 2026-10-27 (unofficial)
