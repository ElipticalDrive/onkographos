# AABB Ray Distance

Also called: Axis-Aligned Bounding Box Intersection Distance

## Overview
Computes the distance along a ray direction from a given point to the nearest intersection with an axis-aligned bounding box (AABB) [S1, unofficial]. It returns -1 if the starting point is inside the box or if the ray points away from the box [S1, unofficial].

## How it works
**Inputs:**
- `point`: The starting position of the ray (3D vector) [S1, unofficial].
- `direction`: The ray direction vector (3D vector) [S1, unofficial].
- `bounding_box_min`: Minimum coordinates of the AABB (3D vector) [S1, unofficial].
- `bounding_box_max`: Maximum coordinates of the AABB (3D vector) [S1, unofficial].

**Outputs:**
- `dist`: The distance to the nearest intersection. Returns -1 if no valid intersection exists (point inside or ray misses) [S1, unofficial].

**Data Structures:**
- Vectors are treated as 3-element arrays or tuples $(x, y, z)$ [G].
- Coordinates are in world space [S1, unofficial].

**Steps:**
1. Normalize the ray direction vector $\vec{d}$ to unit length [S1, unofficial]. Let $\vec{d} = (d_x, d_y, d_z)$.
2. Initialize `dist` to -1 [S1, unofficial].
3. Iterate over each axis $i \in \{x, y, z\}$:
   a. Let $v$ be the coordinate of `point` on axis $i$ [S1, unofficial].
   b. Let $d$ be the component of `direction` on axis $i$ [S1, unofficial].
   c. Let $min_i$ and $max_i$ be the minimum and maximum extents of the AABB on axis $i$ [S1, unofficial].
   d. If `dist` is -1:
      - If $v < min_i$ and $d > 0$: Set `dist` = $(min_i - v) / d$ [S1, unofficial].
      - If $v > max_i$ and $d < 0$: Set `dist` = $(max_i - v) / d$ [S1, unofficial].
   e. Else (`dist` is not -1):
      - If $v < min_i$ and $d > 0$: Set `dist` = $\max(\text{dist}, (min_i - v) / d)$ [S1, unofficial].
      - If $v > max_i$ and $d < 0$: Set `dist` = $\max(\text{dist}, (max_i - v) / d)$ [S1, unofficial].
4. Return `dist` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def aabb_ray_distance(point, direction, bounding_box_min, bounding_box_max):
    # 1. Normalize direction
    length = sqrt(direction[0]**2 + direction[1]**2 + direction[2]**2)
    if length == 0:
        return -1
    d_x = direction[0] / length
    d_y = direction[1] / length
    d_z = direction[2] / length
    
    dist = -1
    
    # Process X axis
    v = point[0]
    d = d_x
    if dist == -1:
        if v < bounding_box_min[0] and d > 0:
            dist = (bounding_box_min[0] - v) / d
        if v > bounding_box_max[0] and d < 0:
            dist = (bounding_box_max[0] - v) / d
    else:
        if v < bounding_box_min[0] and d > 0:
            dist = max(dist, (bounding_box_min[0] - v) / d)
        if v > bounding_box_max[0] and d < 0:
            dist = max(dist, (bounding_box_max[0] - v) / d)
            
    # Process Y axis
    v = point[1]
    d = d_y
    if dist == -1:
        if v < bounding_box_min[1] and d > 0:
            dist = (bounding_box_min[1] - v) / d
        if v > bounding_box_max[1] and d < 0:
            dist = (bounding_box_max[1] - v) / d
    else:
        if v < bounding_box_min[1] and d > 0:
            dist = max(dist, (bounding_box_min[1] - v) / d)
        if v > bounding_box_max[1] and d < 0:
            dist = max(dist, (bounding_box_max[1] - v) / d)
            
    # Process Z axis
    v = point[2]
    d = d_z
    if dist == -1:
        if v < bounding_box_min[2] and d > 0:
            dist = (bounding_box_min[2] - v) / d
        if v > bounding_box_max[2] and d < 0:
            dist = (bounding_box_max[2] - v) / d
    else:
        if v < bounding_box_min[2] and d > 0:
            dist = max(dist, (bounding_box_min[2] - v) / d)
        if v > bounding_box_max[2] and d < 0:
            dist = max(dist, (bounding_box_max[2] - v) / d)
            
    return dist
```

**Parameters:**
- `bounding_box_min/max_x/y/z`: World coordinates defining the AABB extents [S1, unofficial].

**Edge Cases and Pitfalls:**
- The algorithm uses `max` (or `fmaxf`) to keep the largest positive distance when multiple axes yield intersections, effectively finding the entry point into the box [S1, unofficial].
- It returns -1 if the starting point is inside the box or if the ray points away from the box [S1, unofficial].
- Precision issues may arise if the ray direction is very close to zero on an axis; normalization handles this, but division by near-zero direction components should be guarded against in robust implementations [G].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] YueJiang-nj/CVPR2020-SDFDiff, https://github.com/YueJiang-nj/CVPR2020-SDFDiff, fetched 2026-10-27 (unofficial)
