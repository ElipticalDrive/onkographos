# Recursive Ray Tracing

Also called: Higher-order Rays, Secondary Ray Shooting, Whitted shading, Whitted-style Ray Tracing

## Overview
Recursive Ray Tracing extends basic ray casting by tracing secondary rays upon intersection with objects to handle diffuse reflection, specular reflection, and transmission (refraction) [S2, unofficial]. It solves the problem of local illumination and geometric visibility by recursively evaluating the scene, distinguishing itself from its parent [Shadow Ray Casting](2796-shadow-ray-casting.md) by explicitly modeling surface interactions like reflection and refraction rather than just shadow testing [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray (primary or secondary), a list of scene primitives (shapes), a maximum recursion depth, and material properties for each primitive (diffuse, specular, transmission coefficients).
*   **Output:** A color value representing the light contribution at the intersection point.

**Data Structures:**
*   **Ray:** Defined by an origin point $\mathbf{o}$ and a direction vector $\mathbf{d}$.
*   **Intersection:** Contains the hit point $\mathbf{p}$, the surface normal $\mathbf{n}$, the distance $t$ from the ray origin, and a pointer/reference to the intersected primitive.
*   **Primitive/Shape:** Contains geometric data and material properties (e.g., `hasSpecular()`, `hasTransmission()`).

**Numbered Steps:**

1.  **Intersection Test:** Find the closest intersection between the input ray and the scene primitives. If no intersection is found, return the background color [S2, unofficial].
2.  **Shadow Ray Casting (Visibility):** For each light source, construct a shadow ray from the intersection point $\mathbf{p}$ towards the light source. Set the maximum distance (`maxT`) of this shadow ray to the distance between $\mathbf{p}$ and the light source. If this ray intersects any other primitive before reaching the light, the point is in shadow (visibility factor $V_i = 0$); otherwise, $V_i = 1$ [S1, unofficial][S2, unofficial].
3.  **Diffuse Shading:** Calculate the diffuse contribution using the surface normal $\mathbf{n}$, the view direction, and the incident light direction $\mathbf{l}$. The contribution is proportional to the dot product $\mathbf{n} \cdot \mathbf{l}$ and the visibility factor $V_i$ [S1, unofficial][S2, unofficial].
4.  **Specular Reflection:** If the material has specular properties, calculate the ideal reflection direction $\mathbf{r}_{refl}$ using the incident direction $\mathbf{d}_{in}$ and normal $\mathbf{n}$:
    $$ \mathbf{r}_{refl} = 2.0 (\mathbf{n} \cdot \mathbf{d}_{in}) \mathbf{n} - \mathbf{d}_{in} $$
    Trace a new ray from the intersection point in direction $\mathbf{r}_{refl}$ [S2, unofficial].
5.  **Transmission (Refraction):** If the material has transmission properties, calculate the refraction direction $\mathbf{r}_{refr}$ using Snell's law.
    *   Determine if the ray is entering or exiting the object by checking $\cos_i = \mathbf{n} \cdot (-\mathbf{d}_{in})$.
    *   If exiting ($\cos_i \le 0$), flip the normal ($\mathbf{n} = -\mathbf{n}$) and invert the refraction ratio ($\eta = 1.0 / \eta_{initial}$) [S2, unofficial].
    *   Calculate the discriminant $k$:
        $$ k = 1.0 - \eta^2 (1.0 - \cos_i^2) $$
    *   If $k \ge 0$, compute the refracted direction:
        $$ \mathbf{r}_{refr} = \eta \mathbf{d}_{in} + (\eta \cos_i - \sqrt{k}) \mathbf{n} $$
    *   If $k < 0$, total internal reflection occurs; compute specular reflection instead [S2, unofficial].
    *   Trace a new ray from the intersection point in direction $\mathbf{r}_{refr}$ [S2, unofficial].
6.  **Recursion:** The colors returned from the recursive calls for reflection and refraction are combined with the diffuse shading contribution. The recursion continues until the pre-defined maximum depth is reached or the contribution becomes negligible [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def recursive_ray_trace(ray, objects, depth, max_depth, background_color):
    """
    Recursively trace rays to compute color.
    
    Args:
        ray: Ray object with origin and direction.
        objects: List of Shape objects.
        depth: Current recursion depth.
        max_depth: Maximum allowed recursion depth.
        background_color: Color to return if no intersection is found.
        
    Returns:
        Vector3D: The computed color.
    """
    # Step 1: Find closest intersection
    intersection = find_closest_intersection(ray, objects)
    
    # Base case: No intersection
    if intersection is None:
        return background_color
    
    # Base case: Max depth reached
    if depth >= max_depth:
        return Vector3D(0, 0, 0) # Or background, depending on implementation
        
    p = intersection.point
    n = intersection.normal
    shape = intersection.shape
    
    # Step 2 & 3: Diffuse Shading and Shadow Rays
    color = Vector3D(0, 0, 0)
    
    for light in scene.lights:
        # Construct shadow ray
        light_dir = (light.position - p).normalized()
        shadow_ray = Ray(p, light_dir)
        
        # Check occlusion
        shadow_hit = find_closest_intersection(shadow_ray, objects)
        visibility = 1.0
        if shadow_hit is not None:
            # If hit is closer than light, it's in shadow
            # Note: shadow_ray.maxT should be distance to light
            if shadow_hit.distance < (light.position - p).length():
                visibility = 0.0
        
        # Calculate diffuse contribution
        # Assuming simple Lambertian shading for this example
        diffuse_factor = max(0.0, dot(n, light_dir))
        light_intensity = light.intensity
        material_diffuse = shape.material.diffuse_color
        
        color += light_intensity * material_diffuse * diffuse_factor * visibility

    # Step 4: Specular Reflection
    if shape.material.has_specular:
        # Compute reflection direction
        # Note: incident direction for reflection formula is typically -ray.direction
        # if ray.direction points from origin to p. 
        # Standard formula: r = 2(n . -d)n - (-d) = 2(n . -d)n + d
        # However, source [S2, unofficial] uses: reflection = 2.0 * dot(normal, in_light_direction) * normal - in_light_direction
        # where in_light_direction is -(r.d).normalized() in the note, but computeReflectionDirection 
        # in source code takes Direction and normal. 
        # Let's align with Utils::computeReflectionDirection(Direction, normal) which does:
        # 2.0 * dot(normal, Direction) * normal - Direction
        # Here Direction is the incoming ray direction vector.
        
        reflection_dir = compute_reflection_direction(ray.direction, n)
        reflection_ray = Ray(p, reflection_dir)
        reflected_color = recursive_ray_trace(reflection_ray, objects, depth + 1, max_depth, background_color)
        
        # Combine with specular intensity (not explicitly defined in notes, assume 1.0 or material property)
        color += shape.material.specular_intensity * reflected_color

    # Step 5: Transmission (Refraction)
    if shape.material.has_transmission:
        # Default refraction ratio from notes
        eta = 0.75 
        
        # Determine entering/exiting
        # cos_i = dot(-wi, n) where wi is incoming direction (ray.direction)
        cos_i = dot(-ray.direction, n)
        entering = cos_i > 0.0
        
        n_local = n
        eta_local = eta
        
        if not entering:
            # Inside -> Outside
            n_local = -n
            eta_local = 1.0 / eta
            cos_i = dot(-ray.direction, n_local)
            
        # Discriminant
        k = 1.0 - eta_local * eta_local * (1.0 - cos_i * cos_i)
        
        if k >= 0.0:
            # Refraction
            refract_term = sqrt(k)
            first_term = eta_local * ray.direction
            second_term = eta_local * cos_i - refract_term
            refraction_dir = -first_term + n_local * second_term
            
            refraction_ray = Ray(p, refraction_dir)
            refracted_color = recursive_ray_trace(refraction_ray, objects, depth + 1, max_depth, background_color)
            
            # Combine with transmission intensity
            color += shape.material.transmission_intensity * refracted_color
        else:
            # Total Internal Reflection
            # Compute specular reflection instead
            reflection_dir = compute_reflection_direction(ray.direction, n_local)
            reflection_ray = Ray(p, reflection_dir)
            reflected_color = recursive_ray_trace(reflection_ray, objects, depth + 1, max_depth, background_color)
            
            color += shape.material.specular_intensity * reflected_color

    return color

def compute_reflection_direction(direction, normal):
    """
    Computes the perfect reflection direction.
    Formula: 2.0 * dot(normal, direction) * normal - direction
    """
    dot_val = dot(normal, direction)
    return 2.0 * dot_val * normal - direction

def find_closest_intersection(ray, objects):
    """
    Finds the closest intersection between a ray and a list of objects.
    Returns an Intersection object or None.
    """
    closest_t = infinity
    closest_intersection = None
    
    for obj in objects:
        hit = obj.intersect(ray)
        if hit is not None and hit.distance < closest_t:
            closest_t = hit.distance
            closest_intersection = hit
            
    return closest_intersection
```

**Parameters:**
*   **Recursion Depth:** A pre-defined level of recursion limits ray tracing because the contribution to the final pixel color diminishes with each level [S1, unofficial].
*   **Refraction Ratio:** The notes specify a default value of 0.75 for the refraction ratio [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Total Internal Reflection:** If the discriminant $k < 0$ during refraction calculation, total internal reflection occurs. The algorithm must switch to computing specular reflection instead of refraction [S2, unofficial].
*   **Precision Errors:** When tracing secondary rays from the intersection point, floating-point errors can cause the ray to intersect the same surface it just left (self-intersection). An offset (epsilon) should be applied to the ray origin along the normal direction [G].
*   **Infinite Recursion:** Without a maximum depth or a termination condition based on contribution magnitude, rays could theoretically bounce forever in a scene with perfect mirrors [S1, unofficial].

## Subtopics

### Whitted-Style Ray Tracing
*   **What it is:** The specific implementation of recursive ray tracing that handles specular reflection and refraction, named after Turner Whitted.
*   **Differentiation:** It is the standard form of recursive ray tracing for non-diffuse surfaces, as opposed to pure path tracing which handles diffuse surfaces stochastically.
*   **When to choose:** When rendering scenes with glossy or transparent materials (glass, mirrors) where deterministic reflection/refraction is desired.
*   **Link:** [Whitted-Style Ray Tracing](52-whitted-style-ray-tracing.md)

### Path Tracing
*   **What it is:** A stochastic extension of recursive ray tracing that handles diffuse interreflection by sampling the hemisphere.
*   **Differentiation:** While recursive ray tracing (Whitted) only traces reflection/refraction rays deterministically, path tracing traces diffuse bounces probabilistically.
*   **When to choose:** For global illumination effects like color bleeding and soft shadows from area lights, which Whitted-style tracing cannot handle natively.
*   **Link:** [Path Tracing](52-path-tracing.md)

## Operating characteristics
*   **Time Complexity:** Exponential in the worst case with respect to recursion depth, as each reflection/refraction can spawn new rays. In practice, it is bounded by the max depth.
*   **Memory Complexity:** Linear with respect to the recursion depth, as the call stack grows with each recursive step.
*   **Strengths:** Produces highly realistic images with accurate reflections, refractions, and shadows. Simple to implement for basic scenes.
*   **Limitations:** Cannot handle diffuse interreflection (color bleeding) or soft shadows from area lights without extensions. Performance degrades rapidly with high recursion depths and complex scenes.
*   **When to use:** For rendering scenes with transparent or reflective objects where global illumination is not the primary concern.
*   **When not to use:** For scenes requiring realistic diffuse interreflection or soft shadows from large area lights.

## Background
Recursive Ray Tracing was introduced as an extension to basic ray casting to add visual cues like shadows, reflections, and refractions [S1, unofficial]. The term "Whitted shading" refers to the seminal paper by Turner Whitted in 1980, which formalized the use of recursive ray tracing for specular reflection and refraction [S1, unofficial]. The method is widely used in offline rendering and real-time graphics for simulating glossy and transparent materials.

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
