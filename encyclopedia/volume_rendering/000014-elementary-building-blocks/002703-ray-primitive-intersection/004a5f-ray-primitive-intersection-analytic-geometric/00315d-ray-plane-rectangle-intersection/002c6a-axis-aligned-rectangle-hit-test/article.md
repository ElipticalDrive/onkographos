# Axis-Aligned Rectangle Hit Test

Also called: Plane-Rectangle Intersection, XYRect/XZRect/YZRect Hit

## Overview
The Axis-Aligned Rectangle Hit Test determines whether a ray intersects a finite rectangle lying on an axis-aligned plane. It solves the problem of bounding infinite plane intersections to a specific 2D extent, which is essential for modeling flat surfaces in ray tracing. It differs from [Infinite Plane Ray Intersection](5021-infinite-plane-ray-intersection.md) by adding boundary checks for the rectangle's edges, and from [Plane-Cube Intersection](11468-plane-cube-intersection.md) by handling a single planar face rather than a volumetric solid.

## How it works
**Inputs and Outputs:**
*   **Input:** A ray $r$ defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, a minimum distance $t_{min}$, a maximum distance $t_{max}$, and a rectangle object.
*   **Output:** A boolean indicating hit status, and a `HitRecord` structure populated with intersection distance $t$, point $\mathbf{p}$, normal $\mathbf{n}$, and material.

**Data Structures:**
The rectangle is defined by constants bounding two axes and a constant value for the third axis.
*   `XYRect`: Fields `X0`, `X1` (x-bounds), `Y0`, `Y1` (y-bounds), `K` (z-coordinate), `Mat` (material).
*   `XZRect`: Fields `X0`, `X1` (x-bounds), `Z0`, `Z1` (z-bounds), `K` (y-coordinate), `Mat` (material).
*   `YZRect`: Fields `Y0`, `Y1` (y-bounds), `Z0`, `Z1` (z-bounds), `K` (x-coordinate), `Mat` (material).

**Steps:**
1.  **Compute Intersection Parameter ($t$):** Solve the plane equation for the fixed axis. For an `XYRect` (normal along Z), the plane equation is $z = K$. Substituting the ray equation $z(t) = o_z + t \cdot d_z$, we get:
    $$ t = \frac{K - o_z}{d_z} $$
    For `XZRect` (normal along Y): $ t = \frac{K - o_y}{d_y} $.
    For `YZRect` (normal along X): $ t = \frac{K - o_x}{d_x} $.
2.  **Check Depth Bounds:** If $t < t_{min}$ or $t > t_{max}$, the intersection is outside the valid ray segment. Return false.
3.  **Compute Transverse Coordinates:** Calculate the coordinates on the plane's axes at parameter $t$. For `XYRect`:
    $$ x = o_x + t \cdot d_x $$
    $$ y = o_y + t \cdot d_y $$
    (Analogous formulas apply for other orientations).
4.  **Check Boundary Bounds:** Verify if the transverse coordinates lie within the rectangle's extents. For `XYRect`:
    $$ \min(X0, X1) \le x \le \max(X0, X1) \quad \text{and} \quad \min(Y0, Y1) \le y \le \max(Y0, Y1) $$
    If either coordinate is out of bounds, return false.
5.  **Populate HitRecord:**
    *   Set $t_{rec} = t$.
    *   Set $\mathbf{p} = \mathbf{o} + t \cdot \mathbf{d}$.
    *   Determine outward normal $\mathbf{n}_{out}$. For `XYRect`, $\mathbf{n}_{out} = (0, 0, 1)$.
    *   Orient the normal: If $\mathbf{d} \cdot \mathbf{n}_{out} > 0$, the ray hits the back face; negate $\mathbf{n}_{out}$ to ensure it points against the ray direction. Set $\mathbf{n} = \mathbf{n}_{out}$.
    *   Assign material from the rectangle object.

**Pitfalls:**
*   **Division by Zero:** If the ray direction component along the fixed axis is zero (e.g., $d_z = 0$ for `XYRect`), division by zero occurs. The result is `NaN` or `Inf`, which typically fails the $t_{min}/t_{max}$ check, but explicit guards are recommended for robustness [S1, unofficial].
*   **Coordinate Ordering:** Bounds $X0, X1$ are not guaranteed to be sorted ($X0 < X1$). Implementation must use `min` and `max` or ensure input normalization [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hit_xy_rect(rect, ray, t_min, t_max):
    # 1. Compute t for the plane z = rect.K
    denom = ray.direction.z
    if abs(denom) < 1e-9: # Guard against division by zero [G]
        return False
    
    t = (rect.K - ray.origin.z) / denom
    
    # 2. Check depth bounds
    if t < t_min or t > t_max:
        return False
        
    # 3. Compute x and y at t
    x = ray.origin.x + t * ray.direction.x
    y = ray.origin.y + t * ray.direction.y
    
    # 4. Check boundary bounds
    x_min = min(rect.X0, rect.X1)
    x_max = max(rect.X0, rect.X1)
    y_min = min(rect.Y0, rect.Y1)
    y_max = max(rect.Y0, rect.Y1)
    
    if x < x_min or x > x_max or y < y_min or y > y_max:
        return False
        
    # 5. Populate HitRecord
    hit_record.t = t
    hit_record.p = ray.at(t)
    
    # Outward normal for XYRect is +Z
    outward_normal = Vector(0.0, 0.0, 1.0)
    
    # Orient normal against ray direction
    if dot(ray.direction, outward_normal) > 0:
        hit_record.normal = -outward_normal
    else:
        hit_record.normal = outward_normal
        
    hit_record.front_face = (dot(ray.direction, hit_record.normal) < 0)
    hit_record.mat = rect.Mat
    
    return True
```

## Subtopics

### [Axis-Aligned Plane Intersection](12131-axis-aligned-plane-intersection.md)
Intersects a ray with an axis-aligned plane defined by an anchor point, extent, and direction index `d`. It returns the intersection distance `t` and barycentric-like coordinates `u, v`. The method uses a direction index to permute coordinate axes, aligning the plane normal with a single axis to simplify the intersection calculation.

## Operating characteristics
*   **Time Complexity:** $O(1)$. The method involves a constant number of arithmetic operations and comparisons.
*   **Memory Complexity:** $O(1)$. No auxiliary data structures are required beyond the input ray and rectangle definitions.
*   **Strengths:** Extremely fast and numerically stable for axis-aligned primitives. Ideal for bounding boxes and planar surfaces in scene graphs.
*   **Limitations:** Only applicable to axis-aligned rectangles. Requires separate implementations for different orientations (XY, XZ, YZ) or a generalized permutation approach.
*   **When to use:** When rendering flat, axis-aligned surfaces or as part of bounding volume hierarchies (BVH) leaf nodes.
*   **When not to use:** For rotated rectangles or non-planar shapes.

## Background
The method is a standard component in educational ray tracers and simple rendering engines. It is derived from the analytic intersection of a ray with a plane, constrained by rectangular bounds. The specific struct-based implementation shown in the sources is characteristic of modern, object-oriented ray tracer tutorials [S1, unofficial].

## Sources
[S1] hexhowells/GoTracer, https://github.com/hexhowells/GoTracer, fetched 2026-10-26 (unofficial)
