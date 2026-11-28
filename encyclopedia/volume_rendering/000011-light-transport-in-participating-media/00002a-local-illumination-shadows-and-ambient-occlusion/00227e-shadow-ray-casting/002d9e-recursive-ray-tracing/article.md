# Recursive Ray Tracing

Also called: Higher-order Rays, Secondary Ray Shooting, Whitted shading, Whitted-style Ray Tracing

## Overview
Recursive Ray Tracing extends basic ray casting by handling diffuse reflection, specular reflection, and transmission (refraction) through the recursive tracing of rays upon intersection with objects [S2, unofficial]. It solves the problem of limited visual realism in simple ray casting by adding global illumination effects such as shadows, reflections, and refractions, distinguishing itself from its parent [Shadow Ray Casting](2796-shadow-ray-casting.md) by explicitly generating secondary rays for surface interactions rather than just testing visibility for shadowing [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A camera ray (origin $\mathbf{o}$, direction $\mathbf{d}$), a scene description containing a list of geometric primitives, material properties for each primitive (diffuse, specular, transmission coefficients, refractive index $\eta$), and a maximum recursion depth $D_{max}$ [S2, unofficial].
*   **Output:** A final color value $\mathbf{C}$ for the pixel associated with the input ray [G].

**Data Structures:**
*   **Ray:** Defined by origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized) [G].
*   **Intersection:** Contains the hit point $\mathbf{p}$, the surface normal $\mathbf{n}$, the intersected object reference, and the distance $t$ from the ray origin [G].
*   **Material:** Contains flags for specular/transmission properties, color/albedo, and refractive index $\eta$ [G].

**Numbered Steps:**
1.  **Primary Ray Intersection:** Trace the primary ray from the camera into the scene. Find the closest intersection point $\mathbf{p}$ with any primitive. If no intersection is found, return the background color [S2, unofficial].
2.  **Shadow Ray Casting (Visibility):** For each light source, construct a shadow ray from $\mathbf{p}$ towards the light source [S1, unofficial]. The maximum distance for this ray is the distance between $\mathbf{p}$ and the light source [S2, unofficial]. If another primitive intersects this shadow ray before reaching the light, the visibility factor $V_i$ is $0$ (shadow); otherwise, it is $1$ [S1, unofficial][S2, unofficial].
3.  **Diffuse Shading:** Calculate the diffuse contribution using the surface normal $\mathbf{n}$, the view direction, and the incident light direction $\mathbf{l}$ [S2, unofficial]. The contribution is proportional to the dot product $\mathbf{n} \cdot \mathbf{l}$ and the light intensity [S2, unofficial].
4.  **Specular Reflection:** If the material has specular properties, compute the ideal reflection direction $\mathbf{r}$ using the incident direction $\mathbf{d}_{in}$ and normal $\mathbf{n}$:
    $$ \mathbf{r} = 2.0 (\mathbf{n} \cdot \mathbf{d}_{in}) \mathbf{n} - \mathbf{d}_{in} $$
    Trace a new ray in direction $\mathbf{r}$ recursively [S2, unofficial].
5.  **Transmission (Refraction):** If the material has transmission properties, compute the refraction direction using Snell's law [S2, unofficial]. Determine if the ray is entering or exiting the medium by checking $\mathbf{d}_{in} \cdot \mathbf{n}$ [S2, unofficial]. If exiting ($\mathbf{d}_{in} \cdot \mathbf{n} < 0$), flip the normal ($\mathbf{n} = -\mathbf{n}$) and invert the refractive index ratio ($\eta = 1.0 / \eta$) [S2, unofficial].
6.  **Refracted Direction Calculation:** Compute the discriminant $k$:
    $$ k = 1.0 - \eta^2 (1.0 - (\mathbf{d}_{in} \cdot \mathbf{n})^2) $$
    If $k < 0$, total internal reflection occurs; compute the reflection direction instead [S2, unofficial]. If $k \ge 0$, the refracted direction $\mathbf{t}$ is:
    $$ \mathbf{t} = \eta \mathbf{d}_{in} + (\eta (\mathbf{d}_{in} \cdot \mathbf{n}) - \sqrt{k}) \mathbf{n} $$
    Trace a new ray in direction $\mathbf{t}$ recursively [S2, unofficial].
7.  **Recursion and Termination:** Recursively call the ray tracing function for the new rays (reflection/refraction) [S2, unofficial]. The recursion stops if the current depth exceeds $D_{max}$ or if the contribution of the ray becomes negligible [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def trace_ray(ray_origin, ray_direction, scene, depth, max_depth):
    """
    Recursively trace a ray and compute color.
    ray_origin: (x, y, z) tuple
    ray_direction: (x, y, z) tuple, normalized
    scene: dict with keys 'shapes', 'lights', 'bg_color'
    depth: int, current recursion depth
    max_depth: int, maximum allowed recursion depth
    """
    # Base case: max depth reached or no intersection
    intersection = scene.get_closest_intersection(ray_origin, ray_direction)
    if intersection is None or depth >= max_depth:
        return scene.bg_color

    p, normal, material = intersection.point, intersection.normal, intersection.material
    color = [0.0, 0.0, 0.0]

    # 1. Diffuse Shading & Shadow Rays
    for light in scene.lights:
        light_dir = normalize(light.position - p)
        shadow_origin = add(p, scale(normal, 1e-4))
        shadow_ray = (shadow_origin, light_dir)
        shadow_hit = scene.get_closest_intersection(shadow_ray[0], shadow_ray[1], max_t=distance(p, light.position))
        
        visibility = 0.0 if shadow_hit is not None else 1.0
        
        if visibility > 0:
            dot_product = dot(normal, light_dir)
            if dot_product > 0:
                diffuse_term = scale(material.diffuse_color, light.intensity * dot_product)
                color = add(color, scale(diffuse_term, visibility))

    # 2. Specular Reflection
    if material.has_specular:
        reflection_dir = add(scale(normal, 2.0 * dot(normal, ray_direction)), scale(ray_direction, -1.0))
        refl_origin = add(p, scale(normal, 1e-4))
        reflected_color = trace_ray(refl_origin, reflection_dir, scene, depth + 1, max_depth)
        color = add(color, scale(material.specular_color, reflected_color))

    # 3. Transmission (Refraction)
    if material.has_transmission:
        cos_i = dot(scale(ray_direction, -1.0), normal)
        entering = cos_i > 0.0
        
        n = normal
        eta = material.refractive_index
        
        if not entering:
            n = scale(normal, -1.0)
            eta = 1.0 / eta
            cos_i = dot(scale(ray_direction, -1.0), n)
            
        k = 1.0 - eta * eta * (1.0 - cos_i * cos_i)
        
        if k < 0.0:
            refract_dir = add(scale(n, 2.0 * dot(n, ray_direction)), scale(ray_direction, -1.0))
        else:
            refract_dir = add(scale(ray_direction, eta), scale(n, eta * cos_i - math.sqrt(k)))
            
        refract_origin = add(p, scale(n, 1e-4))
        refracted_color = trace_ray(refract_origin, refract_dir, scene, depth + 1, max_depth)
        color = add(color, scale(material.transmission_color, refracted_color))

    return color

def add(a, b): return (a[0]+b[0], a[1]+b[1], a[2]+b[2])
def scale(v, s): return (v[0]*s, v[1]*s, v[2]*s)
def dot(a, b): return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]
def normalize(v):
    l = math.sqrt(dot(v, v))
    return (v[0]/l, v[1]/l, v[2]/l)
def distance(a, b):
    return math.sqrt(dot((a[0]-b[0], a[1]-b[1], a[2]-b[2]), (a[0]-b[0], a[1]-b[1], a[2]-b[2])))
```

**Parameters:**
*   **Recursion Depth ($D_{max}$):** A pre-defined level limiting ray tracing, as contribution diminishes with each level [S1, unofficial]. Recursion may also stop if the contribution is too small [S1, unofficial].
*   **Refractive Index ($\eta$):** Ratio of speeds of light in two media [G]. Default refraction ratio is $0.75$, which is inverted to $1.5$ when the ray exits a medium [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Total Internal Reflection:** Occurs when the discriminant in Snell's law is negative [S2, unofficial]. Must be handled by switching to reflection [S2, unofficial].
*   **Precision Errors:** Intersection points $\mathbf{p}$ may cause self-intersection artifacts [G]. An epsilon offset along the normal is typically required [G].
*   **Infinite Recursion:** If the scene contains mirrors facing each other, recursion may not terminate naturally [G]. A depth limit is a standard stopping condition [S1, unofficial].

## Subtopics

### [Iterative Ray Tracing](7607-iterative-ray-tracing.md)
A GPU-friendly implementation that replaces function calls with an explicit loop to compute ray color, reducing register pressure and shader inlining overhead.

## Operating characteristics
*   **Time Complexity:** Exponential in the worst case ($O(b^d)$ where $b$ is branching factor and $d$ is depth) due to recursive branching for reflections and refractions [G].
*   **Memory Complexity:** Linear in recursion depth $O(d)$ for the call stack [G].
*   **Strengths:** Produces highly realistic images with accurate reflections, refractions, and shadows [G]. Simple to implement for basic scenes [G].
*   **Limitations:** Computationally expensive for deep recursion [G]. Does not handle diffuse inter-reflection (global illumination) natively without extensions like path tracing [G].
*   **When to use:** Realistic rendering of scenes with glossy/specular surfaces and transparent objects [G].
*   **When not to use:** Scenes requiring soft global illumination or complex diffuse bounces (use Path Tracing instead) [G].

## Background
Recursive Ray Tracing was introduced as an extension to basic ray casting to add visual cues like shadows, reflections, and refractions [S1, unofficial]. It is often referred to as Whitted-style ray tracing after Turner Whitted's seminal 1980 paper, which formalized the recursive approach for specular and transparent surfaces [S2, unofficial]. It builds on [Shadow Ray Casting](2796-shadow-ray-casting.md) by adding the generation of secondary rays for surface interactions [S1, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
