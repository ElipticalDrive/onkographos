# Convex Object Optimization

Also called: Convex bounding volume optimization, Inverse ray tracing for convex objects

## Overview
Convex Object Optimization accelerates sphere tracing through objects represented by a signed distance bound enclosed by a convex bounding volume [S1][S2, unofficial]. It solves the problem of expensive interior traversal by avoiding costly marching through the inside of the object, which typically requires many iterations and gradient computations [S1][S2, unofficial]. It sets itself apart from standard sphere tracing by using the bounding volume to determine entry and exit points, thereby reducing the number of iterations required to find a ray-surface intersection [S1][S2, unofficial].

## How it works
The method evaluates the ray origin's position relative to the convex bounding volume and the object to select the appropriate traversal strategy [S1]. 

**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized) [G].
- Object signed distance bound function $f(\mathbf{x})$ [S1].
- Convex bounding volume signed distance bound function $g(\mathbf{x})$ [S1].
- Maximum ray distance $t_{max}$ [G].

**Outputs:**
- Intersection distance $t$ along the ray, or null if no intersection [G].

**Data Structures:**
- No persistent data structures are required [G]. The method relies on evaluating $f$ and $g$ at specific points [S1].

**Steps:**

1. **Evaluate Bounding Volume:** Compute the distance from the ray origin to the bounding volume: $t_g = g(\mathbf{o})$ [S1].
2. **Determine Case:**
   - **Case 1:** $t_g > 0$. The ray origin is outside the bounding volume [S1].
   - **Case 2:** $t_g \le 0$ and $f(\mathbf{o}) > 0$. The ray origin is inside the bounding volume but outside the object [S1].
   - **Case 3:** $t_g \le 0$ and $f(\mathbf{o}) \le 0$. The ray origin is inside both the bounding volume and the object [S1].
3. **Handle Case 1 (Outside Bounding Volume):**
   - Find the intersection of the ray with the bounding volume using sphere tracing [S1].
   - If no intersection with the bounding volume is found within $t_{max}$, return null [G].
   - Set the current position $\mathbf{p} = \mathbf{o} + t_{entry} \mathbf{d}$ [S1].
   - Perform standard sphere tracing using $f(\mathbf{x})$ starting from $\mathbf{p}$ to find the object intersection [S1].
4. **Handle Case 2 (Inside Bounding Volume, Outside Object):**
   - Perform standard sphere tracing using $f(\mathbf{x})$ starting from the ray origin $\mathbf{o}$ to find the object intersection [S1].
5. **Handle Case 3 (Inside Bounding Volume, Inside Object):**
   - Advance the ray to its intersection with the bounding geometry in the forward direction [S1].
   - If no intersection with the bounding volume is found within $t_{max}$, return null [G].
   - Set the starting point for inverse tracing: $\mathbf{p}_{inv} = \mathbf{o} + t_{exit\_bound} \mathbf{d}$ [S1].
   - Perform sphere tracing in the **inverse ray direction** (i.e., direction $-\mathbf{d}$) starting from $\mathbf{p}_{inv}$ using the object function $f(\mathbf{x})$ [S1].
   - The intersection found in the inverse direction corresponds to the object intersection [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def convex_object_optimization(ray_o, ray_d, f, g, t_max):
    # Evaluate bounding volume at origin
    t_g = g(ray_o)
    
    # Case 1: Origin outside bounding volume
    if t_g > 0:
        t_entry_bound = sphere_trace(ray_o, ray_d, g, t_max)
        if t_entry_bound is None:
            return None
        p_start = ray_o + t_entry_bound * ray_d
        t_obj = sphere_trace(p_start, ray_d, f, t_max - t_entry_bound)
        if t_obj is None:
            return None
        return t_entry_bound + t_obj

    # Case 2: Origin inside bounding volume, outside object
    if t_g <= 0 and f(ray_o) > 0:
        t_obj = sphere_trace(ray_o, ray_d, f, t_max)
        return t_obj

    # Case 3: Origin inside bounding volume and inside object
    if t_g <= 0 and f(ray_o) <= 0:
        t_exit_bound = sphere_trace(ray_o, ray_d, g, t_max)
        if t_exit_bound is None:
            return None
        p_inv_start = ray_o + t_exit_bound * ray_d
        t_inv = sphere_trace(p_inv_start, -ray_d, f, t_exit_bound)
        if t_inv is None:
            return None
        return t_exit_bound - t_inv

def sphere_trace(origin, direction, sdf, max_t):
    t = 0.0
    while t < max_t:
        p = origin + t * direction
        dist = sdf(p)
        if dist < 1e-4:
            return t
        t += max(dist, 1e-4)
    return None
```

**Parameters:**
- Threshold for intersection: Not given in the sources. [G] A small epsilon (e.g., $10^{-4}$) is typically used.
- Maximum iterations: Not given in the sources. [G] Determined by `t_max` and step size.

**Edge Cases and Pitfalls:**
- Precision: The inverse trace relies on accurate intersection with the bounding volume. [G] Floating point errors may cause the ray to be slightly outside the bounding volume at the start of the inverse trace.
- Degenerate Input: If the bounding volume is not strictly convex, the sphere tracing may fail or produce incorrect results [S1].
- Tightness: The optimization is most effective when the bounding volume is tight to the object.

## Operating characteristics
- **Time Complexity:** Reduces the number of iterations for Case 3 by avoiding interior traversal of the object SDB [S1][S2, unofficial].
- **Memory Complexity:** No additional memory required [G].
- **Measured Performance:** In a test scene with 3 spheres, a ground plane, and a dodecahedron with a bounding sphere, using two reflection bounces and eight refraction bounces, the convex optimization provided a speedup of around 10% (2.73 ms with optimization vs. 3.03 ms without) [S1][S2, unofficial].
- **Strengths:** Avoids costly marching through the inside of the object, requiring far fewer iterations [S1][S2, unofficial]. Avoids the often costly computation of the gradient per iteration step, which is not feasible in real-time applications [S1][S2, unofficial].
- **Limitations:** Requires a convex bounding volume that encloses the object. The optimization is specific to convex objects [S1].
- **When to use:** Real-time rendering of scene representations with a signed distance bound where objects are convex and enclosed by a convex bounding volume [S1][S2, unofficial].

## Background
The optimization is based on work by Hart et al. [Har96], who proposed various optimizations for sphere tracing convex objects [S1][S2, unofficial]. Most of these optimizations involve the often costly computation of the gradient per iteration step, which is not feasible in real-time applications [S1][S2, unofficial]. The specific method described here focuses on accelerating tracing through an object represented by a signed distance bound enclosed by a convex bounding object [S1][S2, unofficial].

## Sources
[S1] Enhanced Sphere Tracing, https://doi.org/10.2312/stag.20141233, fetched 2026-10-26
[S2] Enhanced Sphere Tracing, https://kev.town/raymarching-toolkit/media/enhanced_sphere_tracing.pdf, fetched 2026-10-27 (unofficial)
