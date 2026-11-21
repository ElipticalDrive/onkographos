# Ray-Octree Boundary Intersection

Also called: intersectOctree

## Overview
Calculates the entry and exit distances of a ray relative to the bounding box of an octree root node [S1, unofficial]. It normalizes the ray direction to be strictly positive to simplify intersection logic and handles zero-direction components to prevent division by zero [S1, unofficial]. This method serves as the core intersection test within the broader [Octree Raycasting](11317-octree-raycasting.md) framework, determining if a ray intersects the octree's spatial bounds before initiating traversal [S1, unofficial]. It distinguishes itself from [Ray-Octant Traversal (Entry)](11321-ray-octant-traversal-entry.md) by operating on the aggregate root bounding box rather than individual octant primitives, providing the initial hit test for the entire structure.

## How it works
**Inputs:**
*   `ray`: A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.
*   `octree`: An octree structure with defined bounds.
*   `flags`: A `RaycastingFlags` object (integer) to store axis flip information.

**Outputs:**
*   An array `[tx0, ty0, tz0, tx1, ty1, tz1]` if the ray hits the octree, where $t_{x0}, t_{y0}, t_{z0}$ are entry distances and $t_{x1}, t_{y1}, t_{z1}$ are exit distances for each axis.
*   `null` if the ray does not intersect the octree.

**Data Structures:**
*   `RaycastingFlags`: An integer where Bit 0 (value 1) indicates Z flip, Bit 1 (value 2) indicates Y flip, and Bit 2 (value 4) indicates X flip [S1, unofficial].

**Steps:**
1.  **Translate Extents:** Translate the octant extents to the scene origin. Set `min` to $(0, 0, 0)$ and `max` to `octree.max - octree.min` [S1, unofficial].
2.  **Calculate Dimensions:** Compute dimensions and half-dimensions of the octree:
    $$ \text{dimensions} = \text{octree.getDimensions()} $$
    $$ \text{halfDimensions} = \text{dimensions} \times 0.5 $$
    [S1, unofficial]
3.  **Translate Origin:** Translate the ray origin to the center of the octant:
    $$ \text{origin} = \text{ray.origin} - \text{octree.getCenter()} + \text{halfDimensions} $$
    [S1, unofficial]
4.  **Reset Flags:** Reset raycasting flags to 0 [S1, unofficial].
5.  **Normalize Direction (Flip):** For each axis ($x, y, z$):
    *   If `direction[axis] < 0`:
        *   Flip the origin coordinate: $\text{origin}[axis] = \text{dimensions}[axis] - \text{origin}[axis]$.
        *   Negate the direction: $\text{direction}[axis] = -\text{direction}[axis]$.
        *   Set the corresponding flag bit: $x \rightarrow 4$, $y \rightarrow 2$, $z \rightarrow 1$ [S1, unofficial].
6.  **Handle Zero Direction:** For each axis, if `direction[axis] == 0`, set `direction[axis]` to `Number.EPSILON` to prevent division by zero [S1, unofficial].
7.  **Compute Inverse Directions:**
    $$ \text{invDir}_x = 1.0 / \text{direction}_x, \quad \text{invDir}_y = 1.0 / \text{direction}_y, \quad \text{invDir}_z = 1.0 / \text{direction}_z $$
    [S1, unofficial]
8.  **Calculate Intersection Parameters:** For each axis, calculate entry ($t_0$) and exit ($t_1$) distances:
    $$ t_{x0} = (\text{min}_x - \text{origin}_x) \times \text{invDir}_x, \quad t_{x1} = (\text{max}_x - \text{origin}_x) \times \text{invDir}_x $$
    (Similarly for $y$ and $z$) [S1, unofficial].
9.  **Check Intersection:**
    $$ \text{hit} = \max(t_{x0}, t_{y0}, t_{z0}) < \min(t_{x1}, t_{y1}, t_{z1}) $$
    Return `[tx0, ty0, tz0, tx1, ty1, tz1]` if hit, else `null` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def intersect_octree(ray_origin, ray_direction, octree_min, octree_max, octree_center, flags):
    # Step 1: Translate extents to scene origin
    min_ext = [0.0, 0.0, 0.0]
    max_ext = [octree_max[i] - octree_min[i] for i in range(3)]
    
    # Step 2: Calculate dimensions and half-dimensions
    dimensions = [max_ext[i] - min_ext[i] for i in range(3)]
    half_dimensions = [d * 0.5 for d in dimensions]
    
    # Step 3: Translate ray origin to center of octant
    origin = [ray_origin[i] - octree_center[i] + half_dimensions[i] for i in range(3)]
    
    # Step 4: Reset flags
    flags.value = 0
    
    # Step 5: Normalize direction (Flip)
    for i in range(3):
        if ray_direction[i] < 0:
            origin[i] = dimensions[i] - origin[i]
            ray_direction[i] = -ray_direction[i]
            # Set flag bit: x=4, y=2, z=1
            if i == 0: flags.value |= 4
            elif i == 1: flags.value |= 2
            elif i == 2: flags.value |= 1
            
    # Step 6: Handle zero direction
    EPSILON = 1e-8 # Number.EPSILON equivalent
    direction = [ray_direction[i] if ray_direction[i] != 0 else EPSILON for i in range(3)]
    
    # Step 7: Compute inverse directions
    inv_dir = [1.0 / direction[i] for i in range(3)]
    
    # Step 8: Calculate intersection parameters
    tx0 = (min_ext[0] - origin[0]) * inv_dir[0]
    tx1 = (max_ext[0] - origin[0]) * inv_dir[0]
    ty0 = (min_ext[1] - origin[1]) * inv_dir[1]
    ty1 = (max_ext[1] - origin[1]) * inv_dir[1]
    tz0 = (min_ext[2] - origin[2]) * inv_dir[2]
    tz1 = (max_ext[2] - origin[2]) * inv_dir[2]
    
    # Step 9: Check intersection
    t_enter = max(tx0, ty0, tz0)
    t_exit = min(tx1, ty1, tz1)
    
    if t_enter < t_exit:
        return [tx0, ty0, tz0, tx1, ty1, tz1]
    else:
        return None
```

**Parameters:**
*   `flags`: A `RaycastingFlags` object used to store axis flip information. Bit 0 (value 1) indicates Z flip, Bit 1 (value 2) indicates Y flip, Bit 2 (value 4) indicates X flip [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero Direction Components:** Rays with zero direction components cause division by zero. The implementation replaces $0.0$ with `Number.EPSILON` [S1, unofficial].

## Subtopics

### [Ray-AABB Intersection for Octree Traversal](13555-ray-aabb-intersection-for-octree-traversal.md)
This algorithm determines whether a ray intersects an axis-aligned bounding box (AABB) representing an octree node and prepares the necessary parameters for traversing its subtrees. It distinguishes itself by normalizing the ray direction to be strictly positive through coordinate flipping, which simplifies the subsequent intersection logic and traversal order determination. This method serves as the core intersection test within the broader [Ray-Octree Boundary Intersection](11318-ray-octree-boundary-intersection.md) framework.

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] sparse-octree module, https://github.com/vanruesc/sparse-octree, fetched 2026-10-27 (unofficial)
