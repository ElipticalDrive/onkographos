# Recursive Ray Tracing

Also called: Higher-order Rays, Secondary Ray Shooting, Whitted shading, Whitted-style Ray Tracing

## Overview
Recursive Ray Tracing extends basic ray casting by handling diffuse reflection, specular reflection, and transmission (refraction) through the recursive tracing of rays upon intersection with objects [S2, unofficial]. It solves the problem of local illumination and shadowing in participating media by evaluating visibility to light sources and simulating surface interactions that influence the final pixel color [S1, unofficial]. It is distinguished from its sibling [Shadow Ray Casting](2796-shadow-ray-casting.md) by explicitly modeling reflection and refraction via physical laws (Snell's law, Fresnel equation) rather than relying solely on shadow tests and ambient approximations [S1, unofficial].

## How it works
**Inputs:** A primary ray $r$ (origin $\mathbf{o}$, direction $\mathbf{d}$), a list of scene objects $O$, a maximum recursion depth $D_{max}$, and a background color function $bg(r)$.
**Outputs:** The final color $C$ of the pixel associated with the ray.

**Data Structures:**
*   **Ray:** Defined by origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized).
*   **Intersection:** Contains hit point $\mathbf{p}$, normal $\mathbf{n}$, distance $t$, and material properties (diffuse color $k_d$, specular color $k_s$, refractive index $\eta$, emissive color $E$).

**Steps:**
1.  **Intersection Test:** Find the closest intersection $I$ between ray $r$ and any object in $O$ such that distance $t > 0$. If no intersection exists, return $bg(r)$.
2.  **Shadow Testing:** For each light source, construct a shadow ray from $\mathbf{p}$ towards the light. Set the shadow ray's maximum distance to the distance between $\mathbf{p}$ and the light. If another primitive intersects this ray before the light, the visibility factor $V_i$ is 0; otherwise, it is 1.
3.  **Diffuse Shading:** Calculate the diffuse contribution using the surface normal $\mathbf{n}$, view direction, and incident light direction $\mathbf{l}$. The contribution is proportional to $\max(0, \mathbf{n} \cdot \mathbf{l})$.
4.  **Specular Reflection:** If the material has specular properties, compute the reflection direction $\mathbf{r}_{dir}$ using the incident direction $\mathbf{v}$ (from surface to eye) and normal $\mathbf{n}$:
    $$ \mathbf{r}_{dir} = 2.0 (\mathbf{n} \cdot \mathbf{v}) \mathbf{n} - \mathbf{v} $$
    Trace a new ray with origin $\mathbf{p}$ and direction $\mathbf{r}_{dir}$.
5.  **Refraction (Transmission):** If the material has transmission properties, compute the refraction direction using Snell's law. Let $\eta$ be the ratio of refractive indices.
    *   Determine if the ray is entering or exiting: if $\mathbf{v} \cdot \mathbf{n} < 0$, the ray is inside the object. Flip the normal ($\mathbf{n} = -\mathbf{n}$) and invert the ratio ($\eta = 1.0 / \eta$).
    *   Calculate $\cos\_angle = \mathbf{v} \cdot \mathbf{n}$.
    *   Calculate the discriminant: $k = 1.0 - \eta^2 (1.0 - \cos\_angle^2)$.
    *   If $k < 0$, total internal reflection occurs; use the reflection direction instead.
    *   If $k \ge 0$, the refracted direction is:
        $$ \mathbf{t}_{dir} = \eta \mathbf{v} + (\eta \cos\_angle - \sqrt{k}) \mathbf{n} $$
        Trace a new ray with origin $\mathbf{p}$ and direction $\mathbf{t}_{dir}$.
6.  **Recursion:** Recursively call the ray tracing function for the new reflection or refraction rays. The recursion depth decreases by 1.
7.  **Accumulation:** Sum the diffuse, specular, and refractive contributions, modulated by visibility and material colors.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trace_ray(r: Ray, objects: List[Shape], depth: int, max_depth: int) -> Color:
    # Base case: max depth reached or no intersection
    if depth <= 0:
        return Color(0, 0, 0)
    
    intersection = get_closest_intersection(r, objects)
    
    if intersection is None:
        return background_color(r)
    
    p, n, t, material = intersection
    
    # Shadow testing
    visibility = 1.0
    for light in lights:
        shadow_ray = Ray(p, light.position - p)
        shadow_hit = get_closest_intersection(shadow_ray, objects)
        if shadow_hit is not None and shadow_hit.distance < (light.position - p).length():
            visibility = 0.0
            break
            
    # Diffuse shading
    diffuse_color = material.diffuse_color * visibility * max(0, dot(n, normalize(light.position - p)))
    
    # Specular reflection
    specular_color = Color(0, 0, 0)
    if material.has_specular:
        view_dir = normalize(camera.position - p)
        # Note: In standard Whitted, reflection is based on incoming ray from eye, 
        # but physically reflection depends on incident light. 
        # Source [S2, unofficial] uses view direction for reflection calculation in specific context, 
        # but standard physics uses incident light. We follow source [S2, unofficial] logic for specular direction if specified, 
        # but standard Whitted uses incident light for reflection. 
        # Source [S2, unofficial] snippet: "compute in_light_direction as -(r.d).normalized()... ideal_reflection = reflection - in_light_direction"
        # This implies reflection of the light ray.
        incident_light = normalize(light.position - p)
        scale_factor = 2.0 * dot(n, incident_light)
        reflection_vec = scale_factor * n - incident_light
        specular_ray = Ray(p, reflection_vec)
        specular_color = material.specular_color * trace_ray(specular_ray, objects, depth - 1, max_depth)
        
    # Refraction
    refract_color = Color(0, 0, 0)
    if material.has_transmission:
        view_dir = normalize(camera.position - p)
        # Determine entering/exiting
        n_local = n
        eta = material.refractive_index
        cos_i = dot(-view_dir, n_local) # view_dir is from p to eye, so -view_dir is from eye to p (incident)
        # Actually, standard refraction uses incident ray from light or eye. 
        # Source [S2, unofficial] uses view_direction for refraction calculation.
        entering = cos_i > 0.0
        
        if not entering:
            n_local = -n_local
            eta = 1.0 / eta
            cos_i = dot(-view_dir, n_local)
            
        k = 1.0 - eta * eta * (1.0 - cos_i * cos_i)
        
        if k < 0.0:
            # Total internal reflection
            scale_factor = 2.0 * dot(n_local, -view_dir) # view_dir is from p to eye
            reflection_vec = scale_factor * n_local - (-view_dir) # Reflect -view_dir
            refract_ray = Ray(p, reflection_vec)
            refract_color = material.transmission_color * trace_ray(refract_ray, objects, depth - 1, max_depth)
        else:
            refract_term = sqrt(k)
            first_term = eta * (-view_dir)
            second_term = eta * cos_i - refract_term
            refract_dir = -first_term + n_local * second_term
            refract_ray = Ray(p, refract_dir)
            refract_color = material.transmission_color * trace_ray(refract_ray, objects, depth - 1, max_depth)
            
    return diffuse_color + specular_color + refract_color
```

**Parameters:**
*   `max_depth`: Pre-defined level of recursion. Typical values are 3-5. Contribution diminishes with each level [S1, unofficial].
*   `refraction_ratio`: Default value 0.75 mentioned in source [S2, unofficial], though physically this is the index of refraction ratio.

**Edge Cases:**
*   **Total Internal Reflection:** Occurs when the discriminant $k < 0$ in refraction calculation. The ray is reflected instead of refracted [S2, unofficial].
*   **Precision Errors:** Intersection tests may suffer from self-intersection artifacts due to floating-point precision. A small epsilon offset is typically added to the ray origin in recursive calls [G].
*   **Infinite Recursion:** Prevented by `max_depth` or by stopping if the contribution is too small [S1, unofficial].

## Subtopics

### Iterative Ray Tracing
A method to compute ray color on GPU by replacing recursive function calls with an iterative loop, because GPU languages like GLSL do not support recursion [S3, unofficial]. It uses an accumulator initialized to 1 and updates it via Hadamard product (element-wise multiplication) with the material color at each bounce. The loop continues until the bounce counter $b < 0$ or no object is hit [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Exponential in the worst case ($O(b^d)$ where $b$ is branching factor and $d$ is depth), but typically linear with respect to the number of primitives per ray for a fixed depth.
*   **Memory Complexity:** Linear with recursion depth $O(d)$ for the call stack. Iterative version uses $O(1)$ stack space.
*   **Strengths:** Produces realistic reflections, refractions, and shadows. Simple to implement for basic scenes.
*   **Limitations:** Does not handle global illumination effects like color bleeding or indirect diffuse lighting (unless extended). Performance degrades rapidly with depth and scene complexity.
*   **When to use:** For offline rendering or real-time applications with limited reflection/refraction requirements and simple scenes.
*   **When not to use:** For scenes requiring full global illumination or very high recursion depths.

## Background
Recursive Ray Tracing was introduced by Turner Whitted in 1980 in his paper "An Improved Illumination Model for Shaded Display" [S1, unofficial]. It extended the basic ray casting algorithm by adding support for specular reflection and refraction, allowing for the rendering of mirrors and glass. The term "higher-order rays" refers to the secondary rays (reflection/refraction) spawned from primary rays [S1, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
[S3] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
