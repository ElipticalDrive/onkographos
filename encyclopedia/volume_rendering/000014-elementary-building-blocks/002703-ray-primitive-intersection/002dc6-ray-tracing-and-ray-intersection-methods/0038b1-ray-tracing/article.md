# Ray Tracing
Also called: Classic ray tracing, Whitted-style Ray Tracing

## Overview
Classic ray tracing (Whitted-style) is a recursive rendering model that captures shadows, specular reflection, refraction, and diffuse surface shading by tracing light paths from the image plane into the scene [S1, unofficial]. It differs from simple ray casting by explicitly spawning secondary rays for reflection and refraction, and shadow rays for occlusion testing, thereby supporting native recursive ray propagation [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scene description containing geometric primitives (surfaces), material properties (reflectivity, refractivity, BRDF), light sources, and a camera model defining an image plane and primary viewing rays.
*   **Output:** A 2D image where each pixel value represents the accumulated radiance along the traced ray path.

**Data Structures:**
*   **Ray:** Defined by an origin point $\mathbf{o}$ and a direction vector $\mathbf{d}$ (normalized).
*   **Intersection Record:** Contains the hit point $\mathbf{p}$, surface normal $\mathbf{n}$, material properties, and distance $t$ from origin.
*   **Scene Graph:** A spatial acceleration structure (e.g., BVH, grid) is implied to efficiently find the closest intersection, though the specific structure is not defined in the core algorithm [G].

**Algorithm Steps:**
The rendering process is recursive. For each pixel, a primary ray is generated. The function `TraceRay(ray, depth)` proceeds as follows:

1.  **Intersection Test:** Find the closest intersection between `ray` and any object in the scene. If no intersection occurs within the scene bounds, return the background color [S1, unofficial].
2.  **Shading (Diffuse):** Calculate the diffuse component of the surface color based on the material's diffuse reflectivity and the illumination from visible light sources.
3.  **Shadow Testing:** For each light source in the scene, cast a **shadow ray** from the intersection point $\mathbf{p}$ toward the light source. If this ray intersects any object before reaching the light, the point is in shadow; otherwise, it is illuminated [S1, unofficial].
4.  **Reflection:** If the material has non-zero reflectivity, spawn a **reflection ray**. The direction $\mathbf{r}$ is calculated using the surface normal $\mathbf{n}$ and incident direction $\mathbf{d}$:
    $$ \mathbf{r} = \mathbf{d} - 2(\mathbf{d} \cdot \mathbf{n})\mathbf{n} $$
    Recursively call `TraceRay(r, depth + 1)`.
5.  **Refraction:** If the material has non-zero refractivity, spawn a **refraction ray**. The direction $\mathbf{t}$ is calculated using Snell's Law, accounting for the indices of refraction of the current medium and the material [S1, unofficial]. Recursively call `TraceRay(t, depth + 1)`.
6.  **Accumulation:** Combine the diffuse, reflected, and refracted contributions according to the material's shading model (e.g., Phong or Blinn-Phong approximation) to produce the final color for the pixel [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trace_ray(ray, scene, max_depth):
    # Step 1: Find closest intersection
    hit = scene.intersect(ray)
    if hit is None:
        return background_color

    # Step 2 & 3: Diffuse shading and Shadow rays
    color = vec3(0.0, 0.0, 0.0)
    for light in scene.lights:
        # Shadow ray from hit point to light
        shadow_ray = Ray(origin=hit.point + hit.normal * epsilon, direction=light.position - hit.point)
        shadow_hit = scene.intersect(shadow_ray)
        
        # If no object blocks the light, add diffuse contribution
        if shadow_hit is None or shadow_hit.distance > (light.position - hit.point).length():
            # Diffuse calculation using BRDF approximation
            diffuse = hit.material.diffuse * max(0.0, dot(hit.normal, light.direction))
            color += diffuse * light.intensity

    # Step 4: Reflection
    if hit.material.reflectivity > 0 and max_depth > 0:
        # Reflect direction: r = d - 2(d.n)n
        reflect_dir = ray.direction - 2 * dot(ray.direction, hit.normal) * hit.normal
        reflect_ray = Ray(origin=hit.point + hit.normal * epsilon, direction=reflect_dir)
        color += trace_ray(reflect_ray, scene, max_depth - 1) * hit.material.reflectivity

    # Step 5: Refraction
    if hit.material.refractivity > 0 and max_depth > 0:
        # Refract direction using Snell's Law (simplified)
        # t = eta * d + (eta * cos(theta_i) - sqrt(1 - eta^2 * sin^2(theta_i))) * n
        # Note: Exact Snell's law implementation requires handling indices of refraction
        # and determining if the ray is entering or exiting the medium.
        refract_dir = compute_refraction(ray.direction, hit.normal, hit.material.ior)
        refract_ray = Ray(origin=hit.point - hit.normal * epsilon, direction=refract_dir)
        color += trace_ray(refract_ray, scene, max_depth - 1) * hit.material.refractivity

    return color
```

**Parameters:**
*   `max_depth`: Maximum recursion depth for secondary rays. Typical values range from 3 to 10 to prevent infinite loops and manage performance [G].
*   `epsilon`: Small offset value added to intersection points to prevent self-intersection artifacts (shadow acne) when spawning secondary rays [G].

**Edge Cases:**
*   **Total Internal Reflection:** When the angle of incidence exceeds the critical angle during refraction, no refracted ray exists; only reflection occurs.
*   **Infinite Recursion:** Caused by perfect mirrors or transparent objects in a closed loop. Mitigated by `max_depth` and energy attenuation per bounce.
*   **Precision Errors:** Floating-point inaccuracies can cause rays to miss intersections or intersect the same surface they originated from. Offset vectors (`epsilon`) are required [G].

## Operating characteristics
*   **Time Complexity:** Exponential in the worst case due to recursion depth and branching factor (reflections/refractions). Average case depends heavily on scene complexity and acceleration structure efficiency.
*   **Memory Complexity:** Linear with respect to the scene graph size plus stack depth for recursion.
*   **Strengths:** Physically plausible results for specular highlights, reflections, and refractions; handles complex lighting interactions natively [S1, unofficial].
*   **Limitations:** Computationally expensive for deep recursion; does not handle diffuse inter-reflection (global illumination) without extensions like path tracing; aliasing artifacts if not supersampled [S2, unofficial].
*   **When to use:** Rendering scenes with significant specular effects, glass, mirrors, or transparent objects where physical accuracy is prioritized over speed [S1, unofficial].
*   **When not to use:** Real-time applications requiring high frame rates without hardware acceleration; scenes dominated by diffuse lighting where simpler rasterization suffices.

## Background
*   **Origin:** First proposed by Turner Whitted in 1980 [S2, unofficial].
*   **History:** Whitted's algorithm extended Appel's earlier ray casting work by introducing recursive secondary rays for reflection and refraction, establishing the foundation for modern ray tracing [S2, unofficial]. It became a standard technique for offline rendering due to its ability to produce photorealistic images with complex lighting [S1, unofficial].
*   **Usage:** Widely used in offline rendering for films, architectural visualization, and product design where image quality is paramount [S1, unofficial].

## Sources
[S1] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S2] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
