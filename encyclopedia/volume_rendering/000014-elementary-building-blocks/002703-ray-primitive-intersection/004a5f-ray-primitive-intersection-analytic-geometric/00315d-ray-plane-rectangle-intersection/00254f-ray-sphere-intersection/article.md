# Ray-Sphere Intersection

Also called: Analytical ray-sphere intersection, Ball Ray Intersection, Quadratic Ray-Sphere Intersection, Ray-Ball Intersection, Ray-Sphere Intersection in Quaternion Space, Sphere-Ray Intersection Check, Sphere Ray Intersection

## Overview
Ray-sphere intersection computes the geometric points where a ray intersects the surface of a sphere, providing the entry and exit distances along the ray parameter $t$ [S6, S7, S9, unofficial]. It solves the problem of efficiently determining if and where a viewing ray hits a spherical primitive, which is a fundamental building block for volume rendering [S1, S6, S7, unofficial]. Unlike [Ray-Triangle Intersection](909-general-ray-triangle-intersection-pl-cker-coordinates.md) which requires barycentric coordinate checks, or [Implicit Surface and SDF Intersection](2795-implicit-surface-and-sdf-intersection.md) which relies on iterative root-finding, this method uses a closed-form analytical solution to the quadratic equation derived from the sphere's implicit definition [S1, S6, S8, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$ (typically normalized); Sphere defined by center $\mathbf{c}$ and radius $r$ [S7, S9, unofficial]. Optional: ray segment bounds $[t_{min}, t_{max}]$ [S1, S2, unofficial].
*   **Output:** Boolean hit status; intersection distance $t$; intersection point $\mathbf{p}$; surface normal $\mathbf{n}$ [S7, S9, unofficial]. For volume rendering, entry and exit points $\mathbf{p}_{entry}, \mathbf{p}_{exit}$ are often required [S6, unofficial].

**Data Structures**
*   **Ray:** Struct containing `origin` (vec3) and `direction` (vec3) [S7, S9, unofficial].
*   **Sphere:** Struct containing `center` (vec3) and `radius` (float) [S2, S5, S7, unofficial].

**Method Steps**
The standard analytical approach transforms the ray into the sphere's local coordinate system (where the sphere is centered at the origin) or solves directly in world space using vector algebra [S1, S6, unofficial].

1.  **Compute Relative Vector:** Calculate the vector from the ray origin to the sphere center: $\mathbf{oc} = \mathbf{o} - \mathbf{c}$ [S1, S6, S8, unofficial].
2.  **Compute Quadratic Coefficients:**
    *   $a = \mathbf{d} \cdot \mathbf{d}$ (Squared length of direction) [S1, S6, S8, unofficial].
    *   $b = 2 (\mathbf{oc} \cdot \mathbf{d})$ [S1, S6, S8, unofficial]. Note: Some implementations use `half_b = oc · d` and adjust the formula accordingly [S2, unofficial].
    *   $c = \mathbf{oc} \cdot \mathbf{oc} - r^2$ [S1, S6, S8, unofficial].
3.  **Compute Discriminant:**
    *   $\Delta = b^2 - 4ac$ [S1, S6, S8, unofficial].
    *   Alternatively, using the geometric projection method: Let $L = \mathbf{c} - \mathbf{o}$. Compute projection $t_{ca} = \mathbf{d} \cdot L$. Compute squared distance from ray line to center $d^2 = |L|^2 - t_{ca}^2$. Then $\Delta = r^2 - d^2$ (scaled by $a$) [S5, unofficial].
4.  **Check Intersection:**
    *   If $\Delta < 0$, the ray misses the sphere. Return miss [S1, S6, S8, unofficial].
    *   If $\Delta = 0$, the ray is tangent. Return hit at $t = -b / (2a)$ [S6, S8, unofficial].
    *   If $\Delta > 0$, there are two roots [S6, S8, unofficial].
5.  **Compute Roots:**
    *   $t_1 = \frac{-b - \sqrt{\Delta}}{2a}$
    *   $t_2 = \frac{-b + \sqrt{\Delta}}{2a}$
    *   Ensure $t_1 \le t_2$ [S6, unofficial].
6.  **Filter Roots:**
    *   If using a ray segment $[t_{min}, t_{max}]$, discard roots outside this range [S1, S2, unofficial].
    *   If $t_1 < t_{min}$, use $t_2$ if $t_2 \ge t_{min}$. Otherwise, miss [S1, S2, unofficial].
    *   If $t_1 \ge t_{min}$, use $t_1$ as the nearest hit [S1, S2, unofficial].
    *   For volume rendering marching bounds, both $t_1$ and $t_2$ are used as entry and exit points, clamped to $[t_{min}, t_{max}]$ and non-negative values [S6, unofficial].
7.  **Compute Output:**
    *   Hit point: $\mathbf{p} = \mathbf{o} + t \mathbf{d}$ [S6, S9, unofficial].
    *   Normal: $\mathbf{n} = \text{normalize}(\mathbf{p} - \mathbf{c})$ [S1, S7, S9, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_sphere_intersection(ray_origin, ray_direction, sphere_center, sphere_radius, t_min=0.0, t_max=float('inf')):
    """
    Computes the intersection of a ray with a sphere.
    
    Args:
        ray_origin: vec3, origin of the ray.
        ray_direction: vec3, direction of the ray (should be normalized).
        sphere_center: vec3, center of the sphere.
        sphere_radius: float, radius of the sphere.
        t_min: float, minimum distance along the ray (default 0.0).
        t_max: float, maximum distance along the ray (default infinity).
        
    Returns:
        tuple: (hit: bool, t: float, point: vec3, normal: vec3)
    """
    # Guard against degenerate ray
    if dot(ray_direction, ray_direction) == 0.0:
        return (False, 0.0, vec3(0), vec3(0))
        
    # 1. Compute relative vector from ray origin to sphere center
    oc = ray_origin - sphere_center
    
    # 2. Compute quadratic coefficients
    a = dot(ray_direction, ray_direction)
    b = 2.0 * dot(oc, ray_direction)
    c = dot(oc, oc) - sphere_radius * sphere_radius
    
    # 3. Compute discriminant
    discriminant = b * b - 4.0 * a * c
    
    # 4. Check if intersection exists
    if discriminant < 0.0:
        return (False, 0.0, vec3(0), vec3(0))
    
    # 5. Compute roots
    sqrt_disc = sqrt(discriminant)
    t1 = (-b - sqrt_disc) / (2.0 * a)
    t2 = (-b + sqrt_disc) / (2.0 * a)
    
    # 6. Filter roots based on ray segment [t_min, t_max]
    # Select the smallest valid t >= t_min
    t_hit = float('inf')
    if t1 >= t_min:
        t_hit = t1
    elif t2 >= t_min:
        t_hit = t2
        
    if t_hit > t_max:
        return (False, 0.0, vec3(0), vec3(0))
        
    # 7. Compute hit point and normal
    hit_point = ray_origin + ray_direction * t_hit
    normal = normalize(hit_point - sphere_center)
    
    return (True, t_hit, hit_point, normal)
```

**Edge Cases and Pitfalls**
*   **Ray Origin Inside Sphere:** If the ray origin is inside the sphere, $c < 0$ and $\Delta > 0$. One root will be negative (behind the origin) and one positive. Standard filtering ($t \ge t_{min}$) handles this, returning the exit point if $t_{min}=0$ [S8, unofficial].
*   **Tangent Ray:** $\Delta = 0$. The ray grazes the sphere. The hit point is valid, but the normal is well-defined [S6, S8, unofficial].
*   **Precision Errors:** Near-tangent rays or large coordinates can cause $\Delta$ to be slightly negative due to floating-point error, resulting in a false miss. A small epsilon tolerance on the discriminant check can mitigate this [G].
*   **Unnormalized Direction:** If $\mathbf{d}$ is not normalized, $a \neq 1$. The formula still holds, but $t$ represents distance scaled by $|\mathbf{d}|^2$. Normalizing $\mathbf{d}$ is recommended for consistent $t$ values [S6, S8, unofficial].
*   **Quaternion Representation:** In quaternion-based formulations, the sphere equation is $\alpha_w(q\bar{q}) + \beta_w q + \bar{q}\bar{\beta}_w + \gamma_w = 0$. Substituting $q(t) = \tau + \omega t$ yields a quadratic in $t$ with coefficients derived from quaternion algebra [S4]. This is mathematically equivalent but computationally more expensive than vector algebra [S4].

## Subtopics

### [3-Sphere Intersection (Trilateration)](5144-3-sphere-intersection-trilateration.md)
Computes the intersection point of three spheres using trilateration, used for generating sample points in sphere carving.

### [Ray-Splat Intersection (Sphere Approximation)](7123-ray-splat-intersection-sphere-approximation.md)
Approximates Gaussian splats as spheres for faster intersection tests, trading geometric accuracy for performance.

### [Spherical Medium Density Integration](9077-spherical-medium-density-integration.md)
Analytically integrates density along a ray segment within a sphere with quadratic falloff, extending intersection to optical depth calculation.

### [Pulsar Forward Pass (Sphere Intersection and Draw)](9396-pulsar-forward-pass-sphere-intersection-and-draw.md)
GPU-accelerated method for differentiable sphere rendering, managing sphere draw info and priority queues for backward passes.

### [Inverse Transforming Spheres](10196-inverse-transforming-spheres.md)
Transforms sphere coefficients into kernel space (inverse Möbius transformation) to enable standard ray-sphere intersection tests on transformed distance fields.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sphere. The method involves a fixed number of arithmetic operations (dot products, square root) [S1, S6, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary memory. No large data structures are required [S1, unofficial].
*   **Performance:** Extremely fast compared to iterative methods like [Raymarching](1010-raymarching.md) or [Ray-Tetrahedron Intersection](2525-ray-tetrahedron-intersection.md). It is the preferred primitive for sphere-based scenes [S6, S9, unofficial].
*   **Strengths:** Exact solution, no convergence issues, minimal branching [S1, S6, unofficial].
*   **Limitations:** Only applicable to spherical primitives. Not suitable for complex geometries without decomposition [S1, unofficial].
*   **When to Use:** When the scene contains spheres or when spheres are used as bounding volumes for other primitives [S1, S6, unofficial].
*   **When Not to Use:** For non-spherical objects or when approximate intersection is acceptable and other primitives are more efficient [S10, unofficial].

## Background
Ray-sphere intersection is a fundamental technique in computer graphics. The analytical solution is derived from substituting the parametric ray equation into the implicit sphere equation [S1, unofficial][S4]. Variations include quaternion-based formulations for conformal geometry [S4] and discrete raymarching approximations for hardware constraints [S11, S12, unofficial].

## Sources
[S1] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
[S2] hexhowells/GoTracer, https://github.com/hexhowells/GoTracer, fetched 2026-10-26 (unofficial)
[S3] Shibodd/volume_path_tracer, https://github.com/Shibodd/volume_path_tracer, fetched 2026-10-26 (unofficial)
[S4] Kernel-Reflection Sequences, https://doi.org/10.2312/stag.20161364, fetched 2026-10-26
[S5] HectorArellanoDev/WebGPUVoxelConeTracing, https://github.com/HectorArellanoDev/WebGPUVoxelConeTracing, fetched 2026-10-26 (unofficial)
[S6] erickTornero/realtime-volumetric-cloudscapes, https://github.com/erickTornero/realtime-volumetric-cloudscapes, fetched 2026-10-27 (unofficial)
[S7] trimesh33/sdf, https://github.com/trimesh33/sdf, fetched 2026-10-27 (unofficial)
[S8] ojaisnielsen/dtu-2010-cuda-volume-rendering, https://github.com/ojaisnielsen/dtu-2010-cuda-volume-rendering, fetched 2026-10-27 (unofficial)
[S9] Ray Marching with Signed Distance Fields - Scribd, https://www.scribd.com/document/831412915/1-Ray-Marching-and-Signed-Distance-Fields, fetched 2026-10-27 (unofficial)
[S10] Volumetric Rendering - Alan Zucconi, https://www.alanzucconi.com/2016/07/01/volumetric-rendering/, fetched 2026-10-30 (unofficial)
