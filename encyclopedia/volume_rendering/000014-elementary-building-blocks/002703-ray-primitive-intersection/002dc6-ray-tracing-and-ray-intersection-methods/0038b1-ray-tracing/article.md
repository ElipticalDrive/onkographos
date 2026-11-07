# Ray Tracing
Also called: Classic ray tracing, Whitted-style Ray Tracing

## Overview
Ray tracing is a rendering model that captures shadows, reflection, refraction, and diffuse surface shading by placing an image plane in the scene and sending rays through each pixel [S1, unofficial]. It differs from simple ray casting by the additional tracing of secondary rays for shadows and refractions [S2, unofficial].

## How it works
**Inputs:** Image plane geometry, camera parameters, scene geometry (primitives), material properties (BRDF, refractive indices), light source positions.
**Outputs:** Color value for each pixel.

**Data Structures:**
*   **Ray:** Defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.
*   **Intersection:** Contains hit point $\mathbf{p}$, surface normal $\mathbf{n}$, material reference, and distance $t$.

**Steps:**
1.  **Primary Ray Generation:** For each pixel in the image plane, generate a primary viewing ray $\mathbf{r}_0$ from the camera center through the pixel center [S1, unofficial].
2.  **Intersection Test:** Find the closest intersection between the ray and scene primitives. If no intersection, return background color [S2, unofficial].
3.  **Shading:** At the intersection point $\mathbf{p}$, calculate the direct lighting contribution.
    *   **Shadow Rays:** For each light source, cast a shadow ray from $\mathbf{p}$ toward the light. If the ray intersects an opaque object before reaching the light, the point is in shadow [S1, unofficial].
    *   **Diffuse Reflection:** Compute shading based on the material's diffuse properties and the visibility determined by shadow rays.
4.  **Secondary Ray Generation (Recursive):**
    *   **Reflection:** If the material is reflective, spawn a reflection ray $\mathbf{r}_{refl}$ based on the incident direction $\mathbf{d}$ and normal $\mathbf{n}$: $\mathbf{r}_{refl} = \mathbf{d} - 2(\mathbf{d} \cdot \mathbf{n})\mathbf{n}$ [S1, unofficial].
    *   **Refraction:** If the material is transparent/translucent, spawn a refraction ray $\mathbf{r}_{refr}$ using Snell's Law, accounting for the indices of refraction of the entering and exiting media [S1, unofficial].
5.  **Recursive Trace:** Trace the secondary rays recursively. The final color is a weighted sum of the direct shading and the colors returned by the recursive calls, weighted by the material's reflectance and transmittance coefficients [S1, unofficial].
6.  **Termination:** Recursion terminates when a maximum depth is reached or the ray energy falls below a threshold.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trace_ray(ray, scene, depth, max_depth):
    # Step 2: Intersection
    hit = scene.intersect(ray)
    if hit is None:
        return background_color
    
    # Step 3: Shading
    # Calculate direct lighting with shadow rays
    direct_light = 0.0
    for light in scene.lights:
        shadow_ray = Ray(hit.point, light.position - hit.point)
        if not scene.intersect(shadow_ray):
            direct_light += compute_diffuse(hit.material, hit.normal, light)
    
    # Step 4: Secondary Rays
    indirect_light = 0.0
    if depth < max_depth:
        # Reflection
        if hit.material.reflective > 0:
            refl_dir = reflect(ray.direction, hit.normal)
            refl_ray = Ray(hit.point, refl_dir)
            indirect_light += hit.material.reflective * trace_ray(refl_ray, scene, depth + 1, max_depth)
        
        # Refraction
        if hit.material.transmissive > 0:
            refr_dir = refract(ray.direction, hit.normal, hit.material.ior)
            refr_ray = Ray(hit.point, refr_dir)
            indirect_light += hit.material.transmissive * trace_ray(refr_ray, scene, depth + 1, max_depth)
            
    return direct_light + indirect_light
```

## Operating characteristics
*   **Complexity:** Time complexity depends on the number of recursive bounces and the cost of intersection tests. Memory complexity is proportional to the recursion depth.
*   **Strengths:** Physically plausible results for specular highlights, reflections, and refractions [S1, unofficial].
*   **Limitations:** Computationally expensive due to recursive ray spawning and multiple intersection tests per pixel [S2, unofficial].
*   **When to use:** When accurate specular effects and transparency are required.
*   **When not to use:** When real-time performance is critical and diffuse shading suffices.

## Background
Ray casting was first presented by Appel [S2, unofficial]. Ray tracing, as defined by the inclusion of secondary rays for shadows and refractions, was first proposed by Whitted [S2, unofficial]. It is considered a classic method for capturing global illumination effects like reflections and refractions [S1, unofficial].

## Sources
[S1] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S2] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
