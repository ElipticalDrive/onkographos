# Orthogonal Culling Optimization
Also called: Orthogonal Culling, Orthogonal Projection Culling

## Overview
Orthogonal Culling is a pre-computation optimization for sphere tracing that reduces the number of signed distance function (SDF) evaluations per ray by determining which scene objects lie within the ray's path before the marching loop begins [S1, unofficial]. It differs from its parent, [Bounding Spheres Optimization](1567-bounding-spheres-optimization.md), which minimizes distance calculations *during* traversal, and its sibling, [Frustum Culling (Sphere vs Frustum)](4835-frustum-culling-sphere-vs-frustum.md), which typically relies on CPU-side broad-phase culling; Orthogonal Culling performs a geometric intersection test between the ray and object bounding volumes using orthogonal projection [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$ (normalized), and a list of scene objects, each defined by a bounding sphere with center $\mathbf{c}$ and radius $r$.
*   **Output:** A boolean mask or list of indices indicating which objects intersect the ray, allowing the renderer to skip SDF evaluations for non-intersecting objects.

**Data Structures**
*   **Ray:** Defined by origin $\mathbf{o} \in \mathbb{R}^3$ and unit direction vector $\mathbf{d} \in \mathbb{R}^3$.
*   **Bounding Sphere:** Defined by center $\mathbf{c} \in \mathbb{R}^3$ and scalar radius $r \ge 0$.

**Algorithm Steps**
1.  For each object in the scene, retrieve its bounding sphere center $\mathbf{c}$ and radius $r$.
2.  Project the vector from the ray origin to the sphere center onto the ray direction to find the closest point on the ray line to the sphere center.
3.  Calculate the squared distance from the sphere center to this closest point on the ray line.
4.  Compare this distance against the squared radius of the bounding sphere.
5.  If the distance is less than the squared radius, the ray intersects the bounding sphere; mark the object as active for this ray.

**Formulas**
Let $\mathbf{v} = \mathbf{c} - \mathbf{o}$ be the vector from the ray origin to the sphere center.
The projection of $\mathbf{v}$ onto the unit direction $\mathbf{d}$ is given by the dot product:
$$ t = \mathbf{v} \cdot \mathbf{d} $$
This $t$ represents the distance along the ray to the closest point $\mathbf{p}$ on the line:
$$ \mathbf{p} = \mathbf{o} + t\mathbf{d} $$
The squared distance $D^2$ from the sphere center $\mathbf{c}$ to the point $\mathbf{p}$ is:
$$ D^2 = \|\mathbf{c} - \mathbf{p}\|^2 = \|\mathbf{c} - (\mathbf{o} + (\mathbf{v} \cdot \mathbf{d})\mathbf{d})\|^2 $$
Alternatively, using the Pythagorean theorem in the triangle formed by $\mathbf{o}$, $\mathbf{c}$, and $\mathbf{p}$:
$$ D^2 = \|\mathbf{v}\|^2 - t^2 = \|\mathbf{c} - \mathbf{o}\|^2 - (\mathbf{v} \cdot \mathbf{d})^2 $$
The intersection condition is:
$$ D^2 < r^2 $$
If this holds, the ray intersects the bounding sphere [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def orthogonal_cull(ray_origin, ray_direction, objects):
    """
    ray_origin: vec3
    ray_direction: vec3 (normalized)
    objects: list of dicts with keys 'center' (vec3) and 'radius' (float)
    returns: list of indices of objects intersecting the ray
    """
    active_objects = []
    
    # Pre-calculate dot product of ray direction with itself (should be 1.0 if normalized)
    # However, for robustness, we use the formula derived from vector projection.
    
    for i, obj in enumerate(objects):
        c = obj['center']
        r = obj['radius']
        
        # Vector from ray origin to sphere center
        v = c - ray_origin
        
        # Projection of v onto ray_direction
        t = dot(v, ray_direction)
        
        # Closest point on the ray line to the sphere center
        # Note: In strict sphere tracing, we might also check t > 0 (in front of camera),
        # but the source notes focus on the geometric intersection of the line and sphere.
        # [G] Standard ray-sphere intersection usually requires t > 0 for forward rays.
        # The source notes imply checking if the object is "in front of the ray" via projection.
        # We assume standard forward-facing rays where t > 0 is a prerequisite or handled by scene bounds.
        
        # Squared distance from sphere center to the closest point on the line
        # D^2 = |v|^2 - t^2
        dist_sq = dot(v, v) - t * t
        
        # Check intersection
        if dist_sq < r * r:
            active_objects.append(i)
            
    return active_objects
```

**Edge Cases and Pitfalls**
*   **Implementation Efficiency:** The current implementation described in the source performs this culling check for *all* objects in the scene for *every* pixel (ray) [S1, unofficial]. This is suboptimal. The optimal approach is to perform the culling once per object to determine which rays it intersects, rather than per ray determining which objects it intersects [S1, unofficial].
*   **Precision:** Floating-point precision errors in the dot product and subtraction may cause false negatives or positives for grazing intersections.
*   **Ray Direction:** If $\mathbf{d}$ is not normalized, the projection formula $t = \mathbf{v} \cdot \mathbf{d}$ yields a scaled value, and the distance calculation $D^2 = \|\mathbf{v}\|^2 - t^2$ becomes invalid. The direction must be normalized [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity**
*   **Time:** $O(N \cdot R)$, where $N$ is the number of objects and $R$ is the number of rays (pixels). The source notes that the current implementation checks all objects per pixel [S1, unofficial].
*   **Memory:** $O(1)$ auxiliary space per ray, assuming the object list is read-only.

**Measured Performance**
*   Tested on a GeForce GTX 1060M at 1920x1080 resolution [S1, unofficial].
*   Performance gain is significant but varies with object count and scene configuration [S1, unofficial].
*   **Table 5.1 Data (FPS):**
    *   1 Object: 600 FPS (optimized) vs 350 FPS (unoptimized)
    *   10 Objects: 500 FPS (optimized) vs 300 FPS (unoptimized)
    *   20 Objects: 40 FPS (optimized) vs 25 FPS (unoptimized)
    *   30 Objects: 29 FPS (optimized) vs 20 FPS (unoptimized)
    *   40 Objects: 6 FPS (optimized) vs 4 FPS (unoptimized)
    *   45 Objects: 4 FPS (optimized) vs 1.5 FPS (unoptimized)
    *   *Note: The source table lists FPS for 1, 10, 20, 30, 40, 45 objects. Intermediate values are not provided.* [S1, unofficial]

**Strengths and Limitations**
*   **Strengths:** Provides a consistent performance increase by reducing the number of SDF evaluations per march step [S1, unofficial]. It is particularly effective when objects are not mutually obstructing (e.g., spheres in a plane), which represents a best-case scenario for the optimization [S1, unofficial].
*   **Limitations:** The naive per-pixel, per-object check is computationally expensive for large $N$. It does not account for occlusion by other objects in the scene, only geometric intersection with the ray [S1, unofficial].

**When to Use**
*   Use when the number of objects is moderate and the cost of SDF evaluation is high relative to the culling cost.
*   Use when objects are spatially distributed such that many rays do not intersect many objects.
*   Do not use if the scene is extremely dense or if the culling overhead exceeds the savings from skipped SDF evaluations.

## Background
**Origin**
Orthogonal Culling was introduced in the context of GPU-based Sphere Tracing implementations to optimize the evaluation of signed distance functions [S1, unofficial]. It was developed as part of a broader set of optimizations including Bounding Spheres and Ray Grouping [S1, unofficial].

**History and Development**
The method was evaluated using a GLSL software shader and simulated on a GPU architecture (C*λ*aSH simulation) [S1, unofficial]. The research highlighted that while the optimization provided high performance gains, the initial implementation was not optimal because it repeated the culling calculation for every ray-object pair per pixel, rather than computing ray-object intersections once per object [S1, unofficial]. The authors suggested that this step could be offloaded to the CPU for better efficiency [S1, unofficial].

**Usage**
The technique is used in real-time ray marching applications, specifically those rendering reflective spheres and mod fields, where reducing the number of distance queries is critical for maintaining high frame rates [S1, unofficial].

## Sources
[S1] Sphere Tracing GPU An evaluation of the Sphere Tracing algorithm and a GPU designed to run it, https://odr.chalmers.se/items/37b0e53f-fc76-4b0e-96d8-8e864573ada9/full, fetched 2026-10-29 (unofficial)
