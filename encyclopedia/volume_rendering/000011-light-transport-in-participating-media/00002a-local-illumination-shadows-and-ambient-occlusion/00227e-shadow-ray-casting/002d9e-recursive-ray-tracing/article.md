# Recursive Ray Tracing

Also called: Higher-order Rays, Secondary Ray Shooting, Whitted shading, Whitted-style Ray Tracing

## Overview
Recursive Ray Tracing extends basic ray casting by handling diffuse reflection, specular reflection, and transmission (refraction) through the recursive tracing of secondary rays upon intersection with objects [S2, unofficial]. It solves the problem of rendering glossy and transparent materials by physically modeling light bounces, distinguishing itself from its parent [Shadow Ray Casting](2796-shadow-ray-casting.md) by adding reflection and refraction branches rather than only shadow visibility tests.

## How it works
**Inputs and Outputs**
*   **Input:** A primary ray $\mathbf{r}$ (origin $\mathbf{o}$, direction $\boldsymbol{\omega}$), a scene description containing geometric primitives and material properties, and a maximum recursion depth $D$.
*   **Output:** The final radiance $L$ (color) for the pixel associated with the primary ray.

**Data Structures**
*   **Ray:** Defined by origin $\mathbf{o}$ and unit direction $\boldsymbol{\omega}$.
*   **Intersection:** Contains hit point $\mathbf{p}$, surface normal $\mathbf{n}$, distance $t$, and material properties (diffuse color, specular coefficient, transmission coefficient, refractive index $\eta$).
*   **Scene:** A list of objects $\mathcal{O}$ supporting intersection queries.

**Numbered Steps**
1.  **Intersection Query:** Find the closest intersection between the current ray $\mathbf{r}$ and the scene $\mathcal{O}$. Let the closest hit be object $o$ at distance $t > 0$. If no intersection exists, return the background color $L_{bg}$ [S2, unofficial].
2.  **Shadow Ray Casting:** For each light source, construct a shadow ray from the intersection point $\mathbf{p}$ towards the light. Calculate the visibility term $V_i(\mathbf{x})$ by checking if any primitive lies between $\mathbf{p}$ and the light source within the distance to the light [S1, unofficial].
3.  **Diffuse Shading:** Compute the diffuse contribution using the surface normal $\mathbf{n}$, view direction, and incident light direction. The contribution is proportional to the light intensity, the material's reflectance, the dot product of the normal and light direction, and the visibility factor [S2, unofficial].
4.  **Specular Reflection:** If the material has specular properties, calculate the ideal reflection direction $\boldsymbol{\omega}_r$.
    *   Compute the scale factor: $s = 2.0 \cdot (\mathbf{n} \cdot \boldsymbol{\omega}_i)$, where $\boldsymbol{\omega}_i$ is the incident direction (typically $-\boldsymbol{\omega}$).
    *   Compute reflection vector: $\mathbf{r}_{vec} = s \cdot \mathbf{n}$.
    *   Ideal reflection direction: $\boldsymbol{\omega}_r = \mathbf{r}_{vec} - \boldsymbol{\omega}_i$ [S2, unofficial].
    *   Recursively trace a ray with origin $\mathbf{p}$ and direction $\boldsymbol{\omega}_r$, incrementing the recursion depth.
5.  **Transmission (Refraction):** If the material has transmission properties, compute the refracted direction $\boldsymbol{\omega}_t$ using Snell's Law [S1, unofficial].
    *   Determine if the ray is entering or exiting the object by checking $\mathbf{n} \cdot \boldsymbol{\omega}$.
    *   If $\mathbf{n} \cdot \boldsymbol{\omega} < 0$ (ray is inside), flip the normal ($\mathbf{n} = -\mathbf{n}$) and invert the refraction ratio ($\eta = 1.0 / \eta$) [S2, unofficial].
    *   Calculate the discriminant: $k = 1.0 - \eta^2 (1.0 - (\mathbf{n} \cdot \boldsymbol{\omega})^2)$ [S2, unofficial].
    *   If $k \ge 0$, compute the refracted direction:
        $$ \boldsymbol{\omega}_t = \eta \boldsymbol{\omega} + (\eta (\mathbf{n} \cdot \boldsymbol{\omega}) - \sqrt{k}) \mathbf{n} $$
    *   If $k < 0$, total internal reflection occurs; compute specular reflection instead [S2, unofficial].
    *   Recursively trace a ray with origin $\mathbf{p}$ and direction $\boldsymbol{\omega}_t$, incrementing the recursion depth.
6.  **Accumulation:** Sum the diffuse, specular, and refractive contributions.
7.  **Recursion Termination:** Stop recursion if the current depth exceeds the maximum depth $D$, or if the contribution falls below a threshold [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trace_ray(ray, objects, max_depth, current_depth):
    # 1. Intersection Query
    intersection = get_closest_intersection(ray, objects)
    
    # Base case: no hit
    if intersection is None:
        return background_color(ray)

    p, normal, material, t = intersection
    
    # Shadow Ray Casting
    visibility = 1.0
    for light in lights:
        shadow_ray = Ray(p, light.position - p)
        shadow_hit = get_closest_intersection(shadow_ray, objects)
        # Check if shadow hit is closer than the light source
        if shadow_hit is not None and shadow_hit.distance < length(light.position - p):
            visibility = 0.0
            break
            
    # 2. Diffuse Shading
    # Assuming standard Lambertian reflection for simplicity as per source logic
    diffuse_color = material.diffuse_color
    light_direction = normalize(lights[0].position - p) # Simplified for single light example
    diffuse_contrib = diffuse_color * visibility * max(0.0, dot(normal, light_direction))
    
    final_color = diffuse_color
    
    # 3. Specular Reflection
    if material.has_specular:
        # Compute reflection direction
        # wi is the incoming ray direction (negative of ray.direction if ray points from eye)
        # Note: Source uses 'Direction' as incident vector for reflection formula
        incident = -ray.direction 
        scale_factor = 2.0 * dot(normal, incident)
        reflection_vec = scale_factor * normal
        reflection_dir = reflection_vec - incident
        
        # Recursive call
        refl_ray = Ray(p, reflection_dir)
        final_color += material.specular_color * trace_ray(refl_ray, objects, max_depth, current_depth + 1)

    # 4. Transmission (Refraction)
    if material.has_transmission:
        eta = material.refractive_index
        incident = -ray.direction # Incident direction relative to surface
        
        # Determine entering/exiting
        cos_i = dot(-incident, normal)
        entering = cos_i > 0.0
        
        if not entering:
            normal = -normal
            eta = 1.0 / eta
            cos_i = dot(-incident, normal)
            
        # Discriminant for Snell's Law
        k = 1.0 - eta * eta * (1.0 - cos_i * cos_i)
        
        if k >= 0.0:
            # Refraction direction
            refract_term = sqrt(k)
            first_term = eta * incident
            second_term = eta * cos_i - refract_term
            refract_dir = -first_term + normal * second_term
            
            # Recursive call
            refract_ray = Ray(p, refract_dir)
            final_color += material.transmission_color * trace_ray(refract_ray, objects, max_depth, current_depth + 1)
        else:
            # Total Internal Reflection
            scale_factor = 2.0 * dot(normal, incident)
            reflection_vec = scale_factor * normal
            refl_dir = reflection_vec - incident
            refl_ray = Ray(p, refl_dir)
            final_color += material.specular_color * trace_ray(refl_ray, objects, max_depth, current_depth + 1)

    return final_color
```

**Parameters**
*   **Recursion Depth ($D$):** A pre-defined level limiting ray tracing because the contribution to the final pixel color diminishes with each level [S1, unofficial]. Typical values are 3-5 for basic scenes.
*   **Refraction Ratio ($\eta$):** The ratio of refractive indices. Default approximation in sources is 0.75 for entering, inverted for exiting [S2, unofficial].

**Edge Cases and Pitfalls**
*   **Precision Errors:** Intersection points $\mathbf{p}$ must be offset slightly along the normal to avoid self-intersection (shadow acne) in recursive rays [G].
*   **Total Internal Reflection:** Handled by checking the discriminant $k < 0$ and switching to reflection [S2, unofficial].
*   **Infinite Recursion:** Must be guarded by `max_depth` or a contribution threshold [S1, unofficial].

## Subtopics

### Iterative Ray Tracing
A method to compute ray color on GPU by replacing recursive function calls with an iterative loop, because GPU languages like GLSL do not support recursion [S3, unofficial]. It uses an accumulator initialized to 1 and updates it via Hadamard product (element-wise multiplication) with the material color at each bounce [S3, unofficial]. It terminates when the bounce counter $b < 0$ or no object is hit [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Exponential in the worst case ($O(b^D)$ where $b$ is the branching factor and $D$ is depth), though typically linear with respect to the number of bounces for fixed depth.
*   **Memory Complexity:** Linear with recursion depth $O(D)$ for the call stack. Iterative variants reduce stack usage but require explicit state management [S3, unofficial].
*   **Strengths:** Physically plausible rendering of glossy and transparent materials; simple to implement for basic scenes.
*   **Limitations:** Does not handle global illumination effects like color bleeding or indirect diffuse lighting without significant extensions (e.g., Metropolis Light Transport) [S1, unofficial]. Computationally expensive for deep recursion.
*   **When to use:** Rendering scenes with mirrors, glass, and glossy surfaces.
*   **When not to use:** Scenes requiring full global illumination or soft shadows from area lights (requires extensions) [S1, unofficial].

## Background
Recursive Ray Tracing was introduced by Turner Whitted in 1980 in his paper "An Improved Illumination Model for Shaded Display" [S1, unofficial]. It extended the basic ray casting algorithm by adding reflection and refraction, allowing for the rendering of realistic mirrors and glass. The term "Whitted-style ray tracing" is commonly used to distinguish this specific recursive approach from modern path tracing or photon mapping techniques.

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
[S3] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
