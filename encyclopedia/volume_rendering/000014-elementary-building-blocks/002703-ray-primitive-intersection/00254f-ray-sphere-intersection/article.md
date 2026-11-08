# Ray-Sphere Intersection
Also called: Analytical ray-sphere intersection, Quadratic Ray-Sphere Intersection, Quaternion-based Ray-Sphere Intersection, Ray-Sphere Intersection in Quaternion Space, Raymarching Sphere Hit Test, Sphere Hit Test, Sphere Ray Intersection, Sphere-Ray Intersection Check

## Overview
Ray-sphere intersection computes the entry and exit points (or the nearest hit distance) where a parametric ray intersects a sphere defined by a center and radius [S5, S6, unofficial]. In volume rendering, this operation is primarily used to determine the marching bounds for ray-marching algorithms, ensuring samples are only taken within the volume's extent [S5, S6, unofficial]. It is distinct from [Sphere Intersection](1620-sphere-intersection.md) in that it solves for the intersection of a line (the ray) with a surface, whereas the latter typically refers to finding points on a sphere's surface satisfying other constraints (such as the 3-sphere trilateration variant) [G]. It is analytically exact and generally faster than iterative methods like [Implicit Surface and SDF Intersection](2795-implicit-surface-and-sdf-intersection.md) for simple spherical primitives [S8, unofficial].

## How it works
**Inputs and Outputs**
*   **Input**: Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$ (typically normalized), sphere center $\mathbf{c}$, sphere radius $r$, and optionally a ray segment $[t_{min}, t_{max}]$ [S1, S5, S6, unofficial].
*   **Output**: Boolean hit status, intersection distance $t$, entry point $\mathbf{p}_{in}$, exit point $\mathbf{p}_{out}$, and surface normal $\mathbf{n}$ [S5, S6, S7, unofficial].

**Data Structures**
The sphere is typically stored as a struct containing:
*   `Center`: A 3D vector $\mathbf{c} = (c_x, c_y, c_z)$ [S2, S5, unofficial].
*   `Radius`: A scalar $r$ [S2, S5, unofficial].
*   `Material`: Optional metadata [S2, unofficial].

**Method Steps**
The standard analytical approach transforms the ray to local space or solves the quadratic equation directly in world space [S1, S6, unofficial].

1.  **Compute Vector $L$**: Calculate the vector from the ray origin to the sphere center:
    $$ \mathbf{L} = \mathbf{c} - \mathbf{o} $$
    [S5, unofficial]

2.  **Compute Quadratic Coefficients**:
    The ray is defined as $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ [G]. Substituting into the sphere equation $|\mathbf{r}(t) - \mathbf{c}|^2 = r^2$ yields $At^2 + Bt + C = 0$ [G].
    *   $A = \mathbf{d} \cdot \mathbf{d}$ (Squared length of direction) [S1, S5, unofficial]. If $\mathbf{d}$ is normalized, $A=1$.
    *   $B = 2 (\mathbf{L} \cdot \mathbf{d})$ [S6, unofficial]. Note: Some implementations use $b = \mathbf{L} \cdot \mathbf{d}$ and adjust the quadratic formula denominator to $A$ instead of $2A$ [S2, unofficial]. We use the standard $B$ form here.
    *   $C = \mathbf{L} \cdot \mathbf{L} - r^2$ [S1, S5, unofficial].

3.  **Compute Discriminant**:
    $$ \Delta = B^2 - 4AC $$
    [S3, S5, unofficial]

4.  **Evaluate Discriminant**:
    *   If $\Delta < 0$, the ray misses the sphere. Return false [S5, unofficial].
    *   If $\Delta = 0$, the ray is tangent. One solution exists.
    *   If $\Delta > 0$, the ray intersects at two points.

5.  **Compute Roots**:
    $$ t_{1,2} = \frac{-B \pm \sqrt{\Delta}}{2A} $$
    [S5, unofficial]
    Ensure $t_1 \le t_2$ by swapping if necessary [S5, unofficial].

6.  **Filter Roots**:
    *   If both $t_1$ and $t_2$ are outside $[t_{min}, t_{max}]$, return false [S1, unofficial].
    *   If $t_1 < t_{min}$, use $t_2$ as the hit distance if it is within bounds, else miss [S1, unofficial].
    *   If $t_1 \ge t_{min}$, use $t_1$ as the entry point [S5, unofficial].
    *   *Note*: Some implementations clamp negative $t$ values to 0.0, effectively treating intersections behind the camera as hits at the origin, which is generally incorrect for strict ray tracing but may be used in specific volume marching contexts [S5, unofficial].

7.  **Compute Output Values**:
    *   Entry point: $\mathbf{p}_{in} = \mathbf{o} + t_1 \mathbf{d}$ (if hit) [S5, unofficial].
    *   Exit point: $\mathbf{p}_{out} = \mathbf{o} + t_2 \mathbf{d}$ (if hit) [S5, S6, unofficial].
    *   Normal at hit point $\mathbf{p}$: $\mathbf{n} = \text{normalize}(\mathbf{p} - \mathbf{c})$ [S6, S8, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_sphere_intersect(ray_origin, ray_direction, sphere_center, sphere_radius, t_min=0.0, t_max=float('inf')):
    # 1. Vector from ray origin to sphere center
    oc = ray_origin - sphere_center
    
    # 2. Quadratic coefficients
    a = dot(ray_direction, ray_direction)
    b = 2.0 * dot(oc, ray_direction)
    c = dot(oc, oc) - sphere_radius * sphere_radius
    
    # 3. Discriminant
    discriminant = b * b - 4.0 * a * c
    
    # 4. Check for intersection
    if discriminant < 0.0:
        return False, None, None, None
        
    sqrt_disc = sqrt(discriminant)
    
    # 5. Roots
    t1 = (-b - sqrt_disc) / (2.0 * a)
    t2 = (-b + sqrt_disc) / (2.0 * a)
    
    # Ensure t1 is the smaller root
    if t1 > t2:
        t1, t2 = t2, t1
        
    # 6. Filter against ray segment
    # If the closest hit is behind t_min, try the further hit
    if t1 < t_min:
        t_hit = t2
    else:
        t_hit = t1
        
    if t_hit > t_max:
        return False, None, None, None
        
    # 7. Compute hit point and normal
    hit_point = ray_origin + ray_direction * t_hit
    normal = normalize(hit_point - sphere_center)
    
    return True, t_hit, hit_point, normal
```

**Edge Cases and Pitfalls**
*   **Ray Origin Inside Sphere**: If $\mathbf{o}$ is inside the sphere, $C < 0$. The discriminant is positive, and one root will be negative (exit behind) and one positive (exit forward). The logic above handles this by selecting the smallest $t \ge t_{min}$. If $t_{min}=0$, it returns the exit point. For volume marching, both entry and exit are often needed; if the ray starts inside, entry is effectively $t_{min}$ or the negative root [S5, unofficial].
*   **Precision Errors**: When the ray is tangent ($\Delta \approx 0$), floating-point errors can cause $\Delta$ to be slightly negative, resulting in a miss. A small epsilon tolerance may be applied to $\Delta$ [S6, unofficial].
*   **Uninitialized Variables**: Incorrect implementations may reference the intersection distance $t$ before it is assigned when computing the normal, leading to garbage values [S6, unofficial].
*   **Direction Normalization**: If $\mathbf{d}$ is not normalized, $A \neq 1$. The quadratic formula must use the computed $A$ [S5, unofficial].

## Subtopics

### [3-Sphere Intersection (Trilateration)](5144-3-sphere-intersection-trilateration.md)
Computes the geometric points where the surfaces of three distinct spheres intersect. Used in sphere carving to generate new sample points on the boundary of a carved volume. It solves a system of three quadratic equations rather than a single ray-sphere intersection.

### [Sphere Intersection](1620-sphere-intersection.md)
General term for finding points on a sphere satisfying specific constraints. In the context of this family, it often refers to point-sphere distance checks or other geometric queries distinct from the ray-parametric intersection.

## Operating characteristics
*   **Time Complexity**: $O(1)$. The method involves a fixed number of arithmetic operations (dot products, square root) regardless of scene size [S5, S8, unofficial].
*   **Memory Complexity**: $O(1)$. Requires storage for a few temporary vectors and scalars [S2, unofficial].
*   **Performance**: Extremely fast. It is one of the fastest primitive intersection tests available, making it suitable for GPU execution in parallel threads [S8, unofficial].
*   **Limitations**: Only applicable to spherical primitives. For complex volumes, the sphere must approximate the volume (e.g., bounding sphere), which can lead to overestimation of the marching distance and wasted samples if the sphere is much larger than the actual volume [S5, S7, unofficial].
*   **When to Use**: When the volume or object is spherical or can be tightly bounded by a sphere. Ideal for early-outs in acceleration structures [S1, unofficial].
*   **When Not to Use**: For non-spherical primitives or when tighter bounds are required (use [Ray-box intersection](776-ray-box-intersection.md) or [Ray-Tetrahedron Intersection](2525-ray-tetrahedron-intersection.md)).

## Background
The analytical ray-sphere intersection is a fundamental technique in computer graphics. It is derived from substituting the parametric line equation into the implicit equation of a sphere [G]. It is widely used in volume rendering for determining marching bounds [S5, unofficial] and in path tracing for shading [S1, S3, unofficial]. The method is often implemented in local space to simplify calculations [S1, unofficial].

## Sources
[S1] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
[S2] hexhowells/GoTracer, https://github.com/hexhowells/GoTracer, fetched 2026-10-26 (unofficial)
[S3] Shibodd/volume_path_tracer, https://github.com/Shibodd/volume_path_tracer, fetched 2026-10-26 (unofficial)
[S4] HectorArellanoDev/WebGPUVoxelConeTracing, https://github.com/HectorArellanoDev/WebGPUVoxelConeTracing, fetched 2026-10-26 (unofficial)
[S5] erickTornero/realtime-volumetric-cloudscapes, https://github.com/erickTornero/realtime-volumetric-cloudscapes, fetched 2026-10-27 (unofficial)
[S6] trimesh33/sdf, https://github.com/trimesh33/sdf, fetched 2026-10-27 (unofficial)
[S7] ojaisnielsen/dtu-2010-cuda-volume-rendering, https://github.com/ojaisnielsen/dtu-2010-cuda-volume-rendering, fetched 2026-10-27 (unofficial)
[S8] Ray Marching with Signed Distance Fields - Scribd, https://www.scribd.com/document/831412915/1-Ray-Marching-and-Signed-Distance-Fields, fetched 2026-10-27 (unofficial)
