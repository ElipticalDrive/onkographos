# Ray-OBB Intersection Test

Also called: OBB slab test, Ray-AABB intersection (OBB variant), Ray-Oriented Bounding Box (OBB) Intersection, Slab method for Oriented Bounding Box

## Overview
The Ray-OBB Intersection Test determines if a ray intersects an Oriented Bounding Box (OBB) and computes the distance to the nearest intersection point [S1, unofficial]. It extends the slab method by projecting the ray onto the OBB's local axes to compute intersection intervals (slabs) along each axis [S1, unofficial]. Unlike [Ray-box intersection](776-ray-box-intersection.md) which assumes axis alignment, this method handles arbitrary orientations by testing intersection with pairs of parallel planes perpendicular to the OBB's local axes [S1, unofficial].

## How it works
**Inputs:**
- Ray origin $\mathbf{O}$ and direction $\mathbf{D}$ [S1, unofficial].
- OBB defined by its world-space position $\mathbf{P}$, local axes $\mathbf{X}, \mathbf{Y}, \mathbf{Z}$ (unit vectors, columns of the transformation matrix), and bounds $aabb\_min[i]$ and $aabb\_max[i]$ along each local axis [S1, unofficial].
- Parallelism threshold $\epsilon = 0.001$ [S1, unofficial].

**Outputs:**
- Boolean indicating intersection [S1, unofficial].
- If intersecting, the distance $t_{min}$ to the nearest intersection point along the ray [S1, unofficial].

**Data Structures:**
- The OBB is represented by a transformation matrix where columns 0, 1, 2 correspond to the X, Y, Z local axes respectively [S1, unofficial].
- The box bounds are defined by minimum and maximum values along each local axis, $aabb\_min[i]$ and $aabb\_max[i]$ [S1, unofficial].

**Steps:**
1. **Initialization:** Set $t_{min} = 0.0$ and $t_{max} = 100000.0$ [S1, unofficial]. Compute the vector from the ray origin to the OBB position: $\mathbf{delta} = \mathbf{P} - \mathbf{O}$ [S1, unofficial].
2. **Iterate over local axes:** For each axis $i \in \{X, Y, Z\}$:
   a. Extract the local axis vector $\mathbf{axis}$ from the transformation matrix columns [S1, unofficial].
   b. Compute the projection of the delta vector onto the axis: $e = \mathbf{axis} \cdot \mathbf{delta}$ [S1, unofficial].
   c. Compute the projection of the ray direction onto the axis: $f = \mathbf{axis} \cdot \mathbf{D}$ [S1, unofficial].
   d. **Non-parallel case:** If $|f| > \epsilon$:
      i. Compute the entry and exit distances for the slab: $t_1 = (e + aabb\_min[i]) / f$ and $t_2 = (e + aabb\_max[i]) / f$ [S1, unofficial].
      ii. Ensure $t_1 \le t_2$ by swapping if necessary [S1, unofficial].
      iii. Update the global interval: $t_{max} = \min(t_{max}, t_2)$ and $t_{min} = \max(t_{min}, t_1)$ [S1, unofficial].
      iv. If $t_{max} < t_{min}$, the ray misses the box; return false [S1, unofficial].
   e. **Parallel case:** If $|f| \le \epsilon$ (ray is parallel to the slab planes):
      i. Check if the ray origin offset lies outside the slab bounds: $(-e + aabb\_min[i] > 0.0)$ OR $(-e + aabb\_max[i] < 0.0)$ [S1, unofficial].
      ii. If either condition is true, the ray is outside the slab; return false [S1, unofficial].
3. **Result:** If the loop completes without early exit, an intersection exists. Set the intersection distance to $t_{min}$ and return true [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_obb_intersection(ray_origin, ray_direction, obb_position, obb_axes, obb_min, obb_max, epsilon=0.001):
    t_min = 0.0
    t_max = 100000.0
    delta = obb_position - ray_origin
    for i in range(3):
        axis = obb_axes[i]
        e = dot(axis, delta)
        f = dot(axis, ray_direction)
        if abs(f) > epsilon:
            t1 = (e + obb_min[i]) / f
            t2 = (e + obb_max[i]) / f
            if t1 > t2:
                t1, t2 = t2, t1
            if t2 < t_max:
                t_max = t2
            if t1 > t_min:
                t_min = t1
            if t_min > t_max:
                return False, None
        else:
            if (-e + obb_min[i] > 0.0) or (-e + obb_max[i] < 0.0):
                return False, None
    return True, t_min
```

**Parameters:**
- $\epsilon = 0.001$: Threshold for determining if the ray is parallel to the OBB axis [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Ray Parallel to Axis:** If the ray direction is parallel to the slab planes, division by zero is avoided by checking $|f| \le \epsilon$. The ray must pass through the slab's extent in the perpendicular direction; otherwise, it misses [S1, unofficial].
- **Sign Conventions:** The parallel case condition $(-e + aabb\_min > 0) \lor (-e + aabb\_max < 0)$ depends on the specific definition of $aabb\_min$ and $aabb\_max$ relative to the axis direction [S1, unofficial].

## Operating characteristics

## Background
The method is derived from the slab method for AABBs, extended to oriented boxes by using the box's local axes for projection [S1, unofficial].

## Sources
[S1] abhishektrip/SliceBasedVolumeRendering, https://github.com/abhishektrip/SliceBasedVolumeRendering, fetched 2026-10-27 (unofficial)
