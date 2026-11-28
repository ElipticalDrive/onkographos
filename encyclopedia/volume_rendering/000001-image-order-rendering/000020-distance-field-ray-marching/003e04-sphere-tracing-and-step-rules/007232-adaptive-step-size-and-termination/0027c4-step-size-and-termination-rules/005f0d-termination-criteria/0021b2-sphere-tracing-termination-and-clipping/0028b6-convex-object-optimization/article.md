# Convex Object Optimization

Also called: Convex Shape Optimization for Internal Origins, Convex bounding volume optimization, Internal Ray Optimization, Inverse ray tracing for convex objects

## Overview
Convex Object Optimization is a technique for accelerating sphere tracing through convex objects represented by a signed distance field (SDF) enclosed by a convex bounding volume. It solves the problem of inefficient ray marching when the ray origin lies inside the object, where standard sphere tracing requires many small steps to traverse the interior. It sets itself apart from standard sphere tracing by avoiding internal steps entirely for internal origins, and from gradient-based optimizations by relying only on the bounding volume geometry rather than per-step gradient computations [S1][S3, unofficial].

## How it works
The method assumes the object is enclosed by a convex bounding volume (e.g., a sphere, box, or convex polyhedron) for which ray intersection can be computed efficiently. It categorizes the ray origin $o$ relative to the bounding volume and the object surface into three cases.

### Inputs and Outputs
- **Inputs**: Ray origin $o$, ray direction $d$ (normalized), object SDF function $f(p)$, bounding volume geometry $B$.
- **Outputs**: Intersection distance $t$ along the ray, or null if no intersection.

### Data Structures
- **Bounding Volume $B$**: A convex shape supporting an analytic ray intersection test. For a sphere, this is defined by center $c$ and radius $r$. For other convex shapes, an intersection function `intersect(ray, B)` returning the smallest positive $t$ is required.
- **SDF $f(p)$**: A function returning the signed distance to the object surface. $f(p) < 0$ indicates inside, $f(p) > 0$ outside.

### Steps

1. **Detect Case**: Determine the position of the ray origin $o$ relative to the bounding volume $B$ and the object $f$.
   - Compute $t_b = \text{intersect}(Ray(o, d), B)$. If $t_b$ is null, the ray does not intersect the bounding volume; return null.
   - Check if $o$ is inside the object: $inside\_obj = (f(o) < 0)$.
   - Check if $o$ is inside the bounding volume: $inside\_bound = (t_b < 0 \text{ or } o \text{ is inside } B)$. Note: Standard ray intersection returns positive $t$ for forward direction. If the origin is inside, the intersection with the boundary in the forward direction is positive, but the origin is geometrically inside. A robust check is: if $f(o) < 0$, the origin is inside the object. If $f(o) \ge 0$, check if $o$ is inside $B$ using a point-in-convex-volume test.

   The three cases are:
   - **Case 1**: Origin outside bounding volume ($o \notin B$).
   - **Case 2**: Origin inside bounding volume but outside object ($o \in B$ and $f(o) \ge 0$).
   - **Case 3**: Origin inside bounding volume and inside object ($o \in B$ and $f(o) < 0$).

2. **Handle Case 1 (Origin outside bounding volume)**:
   - The ray enters the bounding volume at $t_b$.
   - Set new origin $o' = o + t_b \cdot d$.
   - Perform standard sphere tracing starting from $o'$ using $f(p)$ [S1][S3, unofficial].
   - Return the result.

3. **Handle Case 2 (Origin inside bounding volume, outside object)**:
   - The ray is already inside the bounding volume but outside the object.
   - Perform standard sphere tracing starting from $o$ using $f(p)$ [S1][S3, unofficial].
   - Return the result.

4. **Handle Case 3 (Origin inside bounding volume and inside object)**:
   - This is the optimization case. Tracing forward through the interior is costly.
   - Find the intersection of the ray with the bounding volume in the forward direction. Let this distance be $t_{exit\_bound}$.
     - $t_{exit\_bound} = \text{intersect}(Ray(o, d), B)$.
   - Compute the point on the bounding volume boundary: $p_{bound} = o + t_{exit\_bound} \cdot d$.
   - Perform sphere tracing in the **inverse** ray direction starting from $p_{bound}$.
     - Inverse ray: origin $p_{bound}$, direction $-d$.
     - Trace using $f(p)$ until an intersection is found. Let the distance along the inverse ray be $t_{inv}$.
     - The intersection point in world space is $p_{hit} = p_{bound} - t_{inv} \cdot d$.
   - The intersection distance for the original ray is $t = t_{exit\_bound} - t_{inv}$ [S1][S3, unofficial].
   - Return $t$.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def convex_optimized_sphere_trace(o, d, f, B):
    """
    o: vec3, ray origin
    d: vec3, normalized ray direction
    f: function, signed distance field
    B: object, convex bounding volume with intersect(ray) method
    """
    # Step 1: Check if ray intersects bounding volume at all
    t_b = B.intersect(Ray(o, d))
    if t_b is None:
        return None

    # Step 2: Determine if origin is inside the object
    inside_obj = f(o) < 0
    
    # Step 3: Determine if origin is inside the bounding volume
    # Point-in-convex-volume test
    inside_bound = B.contains_point(o)

    # Case 1: Origin outside bounding volume
    if not inside_bound:
        # Advance to bounding volume entry
        o_prime = o + t_b * d
        # Standard sphere trace from entry point
        return sphere_trace(o_prime, d, f)

    # Case 2: Origin inside bounding volume, outside object
    if not inside_obj:
        # Standard sphere trace from origin
        return sphere_trace(o, d, f)

    # Case 3: Origin inside bounding volume and inside object
    # Find exit point from bounding volume
    t_exit_bound = B.intersect(Ray(o, d))
    p_bound = o + t_exit_bound * d
    
    # Trace backwards from boundary
    # Inverse ray: origin p_bound, direction -d
    t_inv = sphere_trace(p_bound, -d, f)
    
    if t_inv is None:
        return None
        
    # Calculate original intersection distance
    # The hit point is p_bound - t_inv * d
    # Original ray: o + t * d = p_bound - t_inv * d
    # o + t * d = (o + t_exit_bound * d) - t_inv * d
    # t * d = (t_exit_bound - t_inv) * d
    # t = t_exit_bound - t_inv
    t = t_exit_bound - t_inv
    
    return t

def sphere_trace(o, d, f):
    """
    Standard sphere tracing loop.
    """
    max_steps = 1000
    max_dist = 100.0
    t = 0.0
    
    for i in range(max_steps):
        p = o + t * d
        dist = f(p)
        
        if dist < 1e-4: # Hit threshold
            return t
        if t > max_dist:
            return None
            
        t += dist
        
    return None
```

### Parameters
- **Hit Threshold**: $10^{-4}$ is typical for sphere tracing [S1].
- **Max Steps**: Not given in the sources. [G] A value of 1000 is standard for sphere tracing.
- **Max Distance**: Not given in the sources. [G] Scene-dependent.

### Edge Cases and Pitfalls
- **Precision**: When $t_{exit\_bound} \approx t_{inv}$, the result $t$ may suffer from cancellation errors. However, since $t_{inv}$ is the distance from the boundary to the surface, and the origin is inside, $t_{inv}$ is generally small compared to $t_{exit\_bound}$ unless the origin is very close to the boundary.
- **Degenerate Bounding Volume**: If the bounding volume is not strictly convex, the intersection test may fail or return incorrect results.
- **Origin on Boundary**: If $f(o) \approx 0$, the case classification may be unstable. [G] Treat as outside object (Case 2) if $f(o) \ge 0$.

## Operating characteristics
- **Time Complexity**: Reduces the number of iterations for internal origins from $O(D)$ where $D$ is the diameter of the object, to $O(1)$ for the bounding volume intersection plus $O(d_{exit})$ where $d_{exit}$ is the distance from the boundary to the surface. This is significantly faster for large objects.
- **Memory Complexity**: $O(1)$ additional memory for the bounding volume intersection.
- **Measured Performance**: In a test scene with 3 spheres, a ground plane, and a dodecahedron with a bounding sphere, using two reflection bounces and eight refraction bounces, the convex optimization provided a speedup of around 10% (2.73 ms with optimization vs. 3.03 ms without) [S1].
- **Strengths**: Avoids costly marching through the interior of the object. Avoids the often costly computation of the gradient per iteration step, which is not feasible in real-time applications [S1][S3, unofficial].
- **Limitations**: Requires a convex bounding volume that supports efficient ray intersection. Not applicable to non-convex objects without decomposition.

## Background
The optimization was proposed by Hart et al. [Har96] for sphere tracing convex objects. However, many of the original optimizations involved the often costly computation of the gradient per iteration step, which is not feasible in real-time applications [S1][S3, unofficial]. The specific implementation described here, which uses inverse ray tracing from a bounding volume, is presented in the context of Enhanced Sphere Tracing [S1][S3, unofficial].

## Sources
[S1] Enhanced Sphere Tracing, https://doi.org/10.2312/stag.20141233, fetched 2026-10-26
[S2] 要約EnhancedSphereTracing～より良いレイマーチングに向けて～, https://qiita.com/Hirai0827/items/eddcb73a1976c3088b88, fetched 2026-10-26 (unofficial)
[S3] Enhanced Sphere Tracing, https://kev.town/raymarching-toolkit/media/enhanced_sphere_tracing.pdf, fetched 2026-10-27 (unofficial)
