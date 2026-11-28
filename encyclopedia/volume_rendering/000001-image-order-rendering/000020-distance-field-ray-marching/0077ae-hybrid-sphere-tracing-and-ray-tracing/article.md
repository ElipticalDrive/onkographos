# Hybrid Sphere Tracing and Ray Tracing
Also called: Hybrid Ray Tracing and Sphere Tracing

## Overview
Hybrid sphere tracing and ray tracing is a rendering technique that accelerates Signed Distance Function (SDF) evaluation by using exact ray tracing for primitives with closed-form intersections or BVH traversability, while falling back to sphere tracing for sub-expressions lacking a ray form, such as smooth blends or offsets [S1, unofficial]. It distinguishes itself from pure sphere tracing by eliminating the iterative closest-point search for standard primitives, and from pure ray tracing by retaining the flexibility of SDF-based CSG operations and procedural shapes that do not have analytic ray intersection functions [S1, unofficial].

## How it works
The method combines two traversal strategies based on the geometric properties of the scene primitives. It operates on a scene defined by a composition of primitives, CSG operations, and affine transformations, represented as a "tape" or expression tree [S1, unofficial].

**Inputs and Outputs**
*   **Input**: A ray $r(t) = \mathbf{o} + t\mathbf{d}$, where $\mathbf{o}$ is the origin and $\mathbf{d}$ is the normalized direction vector. A scene graph containing primitives (analytic shapes, mesh SDFs), CSG operators, and procedural modifiers.
*   **Output**: The hit distance $t$, surface normal $\mathbf{n}$, and material properties, or a miss if the ray escapes.

**Data Structures**
*   **Ray Interval List**: For CSG operations, each primitive returns a list of intervals $[t_{enter}, t_{exit}]$ along the ray where the ray is inside the primitive. To satisfy device constraints (no dynamic allocation, no recursion), these lists have a fixed maximum length per ray [S1, unofficial].
*   **Bounding Boxes**: Each primitive or sub-expression has an axis-aligned bounding box (AABB). For mesh SDFs, these are the leaf boxes of the BVH. For procedural modifiers (e.g., Offset), the box is grown by the modifier's distance [S1, unofficial].

**Algorithm Steps**
1.  **Primitive Classification**: Determine if the current primitive or sub-expression supports a closed-form ray intersection or BVH traversal.
    *   **Ray-Form Primitives**: Analytic shapes (sphere, box, cylinder, cone, torus) and mesh SDFs.
    *   **Sphere-Trace Primitives**: Smooth blends, Offset, Annular, Blur, Mollify, Elongate, Repeat, and NotADistance shapes [S1, unofficial].
2.  **Ray Query for Ray-Form Primitives**:
    *   For analytic shapes, compute the intersection analytically.
    *   For mesh SDFs, perform a BVH ray traversal using Möller–Trumbore tests against `TriangleSoA` leaves. This uses the existing packed BVH without building a new acceleration structure [S1, unofficial].
    *   Return the hit distance $t$, normal $\mathbf{n}$, and entry/exit status.
3.  **CSG Combination via Ray Intervals**:
    *   For CSG operations (Union, Intersection, Difference, Complement), combine the ray intervals from child primitives.
    *   Union: Merge intervals.
    *   Intersection: Intersect intervals.
    *   Difference: Subtract child intervals from parent.
    *   Complement: Invert intervals.
    *   Affine maps transform the ray $\mathbf{o}, \mathbf{d}$ instead of the query point, ensuring exactness without distance evaluation [S1, unofficial].
4.  **Sphere Tracing for Non-Ray-Form Sub-Expressions**:
    *   If a sub-expression (e.g., a smooth blend) has no ray form, sphere trace only within its bounding box.
    *   Initialize $t$ to the entry distance of the bounding box.
    *   Evaluate the SDF $f(\mathbf{p})$ at $\mathbf{p} = \mathbf{o} + t\mathbf{d}$.
    *   Step: $t \leftarrow t + f(\mathbf{p})$.
    *   Terminate if $f(\mathbf{p}) < \epsilon$ (hit) or $t$ exceeds the box exit distance (miss) [S1, unofficial].
5.  **Shading**: Compute lighting, shadows, and ambient occlusion. Shadows and AO use short ray queries. Soft shadows may require multiple shadow rays or sphere-traced penumbra [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hybrid_trace(ray, scene, max_intervals=64):
    """
    ray: struct { origin: vec3, direction: vec3 }
    scene: Primitive or CSGNode or ProceduralNode
    max_intervals: fixed size for interval lists
    """
    # 1. Check if primitive supports ray query
    if scene.supports_ray_query():
        # 2. Perform exact ray query
        hit = scene.ray_intersect(ray)
        if hit.hit:
            return hit
        else:
            # If it's a CSG node, combine intervals
            if scene.is_csg():
                intervals = combine_intervals(scene.children, ray, max_intervals)
                return process_intervals(intervals, ray)
            else:
                return MISS

    # 3. Fall back to sphere tracing for procedural/non-ray forms
    # Get bounding box of the sub-expression
    bbox = scene.bounding_box()
    
    # Check if ray intersects bounding box
    t_min, t_max = ray_box_intersect(ray, bbox)
    if t_min > ray.max_t or t_max < ray.min_t:
        return MISS

    # Initialize sphere trace within the box
    t = max(t_min, ray.min_t)
    
    while t < t_max and t < ray.max_t:
        p = ray.origin + t * ray.direction
        d = scene.sdf(p)
        
        if d < EPSILON:
            return HIT(t, p, compute_normal(scene, p))
        
        # Step forward
        t += d
        
        # Safety check to prevent infinite loops in degenerate cases
        if t > ray.max_t:
            return MISS
            
    return MISS

def combine_intervals(children, ray, max_len):
    """
    Combines ray intervals for CSG operations.
    Returns a fixed-size list of intervals.
    """
    all_intervals = []
    for child in children:
        if child.supports_ray_query():
            hit = child.ray_intersect(ray)
            if hit.hit:
                all_intervals.append([hit.t, hit.t + hit.depth]) # Simplified
            else:
                # For CSG, we need entry/exit intervals
                # This is a placeholder for the actual interval logic
                pass
        else:
            # For non-ray forms, we might need to sphere trace to find intervals
            # or use bounding box approximations
            pass
            
    # Sort and merge intervals based on CSG operator
    # Union: merge overlapping
    # Intersection: keep only overlaps
    # Difference: subtract
    return merge_intervals(all_intervals, max_len)
```

**Parameters**
*   **Maximum Interval List Length**: Fixed to stay within device constraints (no allocation, no recursion). Typical values are not specified, but must be sufficient for the complexity of CSG operations [S1, unofficial].
*   **Mesh Leaf Triangle Count**: In tested scenes, mesh leaves were capped at $\le 128$ triangles to balance BVH traversal cost and sphere tracing fallback [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Sharp CSG**: For sharp CSG, the surface lies on the primitives' surfaces. Bounding boxes must cover the mesh's leaf boxes plus any analytic shapes (like cylinders) where the surface is deep inside the mesh and not reached by leaf boxes [S1, unofficial].
*   **Procedural Growth**: Smooth blends and offsets grow the bounding boxes, potentially increasing the sphere tracing volume [S1, unofficial].
*   **Performance**: Sphere tracing near the surface requires many closest-point searches, especially for grazing rays, which can be costly [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Hybrid is approximately 2x faster than pure sphere tracing for mesh scenes. However, pure ray tracing is significantly faster (e.g., 17 ms vs 1006 ms for sphere tracing in the tunnel scene) [S1, unofficial].
*   **Memory Complexity**: Fixed-size interval lists per ray to avoid dynamic allocation. No new acceleration structures for mesh SDFs, reusing existing BVHs [S1, unofficial].
*   **Measured Performance**:
    *   **Scene**: Tunnel (960x960, 2x2 SS, RTX 3080 Ti) [S1, unofficial].
    *   **Sphere Tracing**: 1006 ms.
    *   **Hybrid (mesh leaves ≤ 128)**: 488 ms.
    *   **Pure Ray Tracing**: 17 ms.
    *   **Correctness**: Hybrid agrees with pure ray tracing to 39 dB PSNR. Differing pixels (2%) are due to shading differences (vertex normals vs distance-gradient normals) [S1, unofficial].
*   **Strengths**: Retains SDF flexibility for procedural shapes while accelerating standard primitives. Device-capable from the start.
*   **Limitations**: Not as fast as pure ray tracing for simple primitives. Sphere tracing fallback can be slow for grazing rays or complex procedural shapes.

## Background
The method was proposed in the context of the EBGeometry library, aiming to optimize sphere-traced rendering of shapes and tapes [S1, unofficial]. It addresses the performance bottleneck of sphere tracing for mesh SDFs by leveraging existing BVH structures and exact ray queries where possible [S1, unofficial].

## Sources
[S1] Visualization: sphere-traced rendering of shapes and tapes, https://github.com/rmrsk/EBGeometry/issues/93, fetched 2026-10-26 (unofficial)
