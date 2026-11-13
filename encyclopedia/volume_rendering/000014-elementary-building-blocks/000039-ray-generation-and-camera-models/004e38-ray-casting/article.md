# Ray Casting
Also called: Primary Ray Tracing

## Overview
Ray casting is the fundamental image synthesis technique that determines the color of each pixel by casting primary rays from a camera through the image plane into the scene and computing the first intersection with visible geometry [S1, S2, S3, unofficial]. It differs from full ray tracing by excluding secondary rays for shadows, reflections, or refractions, serving as the base visibility computation upon which advanced effects are built [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** A camera model defining the origin $\mathbf{o}$ and direction $\mathbf{d}$ for each pixel; a scene representation containing geometric primitives (e.g., spheres, triangles) with their surface equations [S2, S3, unofficial].
*   **Outputs:** A 2D image where each pixel $(u, v)$ contains the color computed at the closest intersection point along the corresponding ray [S3, unofficial].

**Data Structures**
*   **Ray:** Defined by origin $\mathbf{o}$ and direction $\mathbf{d}$. The direction is typically normalized [G].
*   **Primitive:** Stores geometric parameters (e.g., sphere center $\mathbf{c}$ and radius $r$) and a function to compute intersection distance $t$ [S3, unofficial].

**Algorithm Steps**
1.  **Ray Generation:** For each pixel in the image, generate a primary ray $R(t)$ using the camera model [S2, unofficial].
2.  **Intersection Testing:** For each primitive in the scene, calculate the intersection parameter $t$ between the ray and the primitive [S3, unofficial].
3.  **Closest Hit Selection:** Identify the primitive with the smallest positive $t$ (closest to the camera origin) [S3, unofficial].
4.  **Color Computation:** Compute the color at the intersection point $\mathbf{p} = R(t_{min})$ based on the primitive's material properties or distance [S3, unofficial].

**Formulas**
The ray equation is defined as [S2, S3, unofficial]:
$$ R(t) = \mathbf{o} + t \mathbf{d} $$
where $\mathbf{o}$ is the ray origin, $\mathbf{d}$ is the direction vector, and $t$ is the scalar distance along the ray.

For a sphere with center $\mathbf{c}$ and radius $r$, the intersection is found by solving for $t$ in [S3, unofficial]:
$$ |\mathbf{o} + t \mathbf{d} - \mathbf{c}|^2 = r^2 $$
Expanding this yields a quadratic equation [S3, unofficial]:
$$ t^2 (\mathbf{d} \cdot \mathbf{d}) + 2t (\mathbf{d} \cdot (\mathbf{o} - \mathbf{c})) + (\mathbf{o} - \mathbf{c}) \cdot (\mathbf{o} - \mathbf{c}) - r^2 = 0 $$
Let $A = \mathbf{d} \cdot \mathbf{d}$, $B = 2 \mathbf{d} \cdot (\mathbf{o} - \mathbf{c})$, and $C = (\mathbf{o} - \mathbf{c}) \cdot (\mathbf{o} - \mathbf{c}) - r^2$. The solutions are [S3, unofficial]:
$$ t = \frac{-B \pm \sqrt{B^2 - 4AC}}{2A} $$
If the discriminant $B^2 - 4AC < 0$, there is no intersection. The smallest positive $t$ is selected [S3, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_cast(scene, camera, width, height):
    image = empty array of size (width, height)
    
    for y from 0 to height - 1:
        for x from 0 to width - 1:
            # Generate primary ray for pixel (x, y)
            # Implementation depends on camera model [G]
            ray_origin = camera.get_origin(x, y)
            ray_direction = camera.get_direction(x, y)
            ray_direction = normalize(ray_direction)
            
            closest_t = infinity
            hit_primitive = None
            
            # Enumerate primitives along ray
            for primitive in scene.primitives:
                # Compute intersection distance t
                t = primitive.intersect(ray_origin, ray_direction)
                
                # Check if intersection is valid and closer than current best
                if t > 0 and t < closest_t:
                    closest_t = t
                    hit_primitive = primitive
            
            # Compute color at hit point
            if hit_primitive is not None:
                hit_point = ray_origin + closest_t * ray_direction
                image[y, x] = hit_primitive.compute_color(hit_point)
            else:
                image[y, x] = background_color
                
    return image
```

**Edge Cases and Pitfalls**
*   **Degenerate Rays:** If $\mathbf{d}$ is not normalized, the quadratic coefficients change, potentially leading to incorrect $t$ values if not handled consistently [S3, unofficial].
*   **No Intersection:** If no primitive intersects the ray, the pixel is colored with the background [S3, unofficial].
*   **Precision:** Floating-point errors in the discriminant calculation can cause false negatives for grazing intersections [G].

## Subtopics

### [Ray Sample Generation](12271-ray-sample-generation.md)
Generates multiple rays per pixel for anti-aliasing.

### [Grid Ray Sampler](3930-grid-ray-sampler.md)
Optimizes ray traversal for structured volume grids.

### [Compute Shader Ray Marching Setup](3487-compute-shader-ray-marching-setup.md)
Adapts ray casting for GPU parallel execution in volume rendering.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of pixels and $M$ is the number of primitives, assuming brute-force intersection testing [S3, unofficial].
*   **Memory Complexity:** $O(N)$ for the output image, plus storage for scene primitives [S2, unofficial].
*   **Strengths:** Simple implementation; produces correct visibility for opaque objects; serves as the foundation for more complex rendering techniques [S1, S3, unofficial].
*   **Limitations:** Does not handle transparency, shadows, or reflections inherently; performance degrades linearly with scene complexity without acceleration structures [S1, S3, unofficial].
*   **When to Use:** Basic visibility checks; educational purposes; as a base for ray marching in volume rendering where analytical intersections are not feasible [S1, S3, unofficial].

## Background
Ray casting was introduced as the basic form of ray tracing, considering only primary rays without secondary rays for shadows, reflections, or refractions [S1, unofficial]. It is a physically-based image synthesis technique that generates primary rays through pixels to determine visible objects and accumulates contributions along the ray to compute light distribution [S2, unofficial]. The method is widely used in computer graphics for its simplicity and correctness in determining visibility [S3, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
[S3] http://www.hugi.scene.org/online/hugi37/hugi%2037%20-%20coding%20adok%20on%20ray%20casting,%20ray%20tracing,%20ray%20marching%20and%20the%20like.htm, http://www.hugi.scene.org/online/hugi37/hugi%2037%20-%20coding%20adok%20on%20ray%20casting,%20ray%20tracing,%20ray%20marching%20and%20the%20like.htm, fetched 2026-10-26 (unofficial)
