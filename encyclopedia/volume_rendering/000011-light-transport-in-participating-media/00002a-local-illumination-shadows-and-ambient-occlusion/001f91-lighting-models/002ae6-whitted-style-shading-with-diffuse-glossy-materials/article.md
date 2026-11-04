# Whitted-Style Shading with Diffuse/Glossy Materials

Also called: Diffuse Reflection, Lambertian reflection, Whitted-Style Shading

## Overview
Whitted-style shading with diffuse/glossy materials computes the color of a ray intersection by combining ambient lighting with direct illumination from light sources, using a simplified reflectance model for surfaces [S2, unofficial]. It differs from full path tracing by limiting recursion to a single bounce (or fixed depth) and using analytical approximations for specular/glossy reflections rather than stochastic sampling, and it differs from pure local illumination models by explicitly casting shadow rays to determine visibility [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A ray $\vec{r}$ (origin $\vec{o}$, direction $\vec{d}$), a scene containing a list of objects (`objList`) and light sources (`LightSourceList`), and material properties for each object.
- Output: A color value (RGB) representing the radiance at the intersection point.

**Data Structures:**
- `Ray`: Contains origin $\vec{o}$, direction $\vec{d}$, minimum distance `minT`, maximum distance `maxT`, and recursion `depth` [S2, unofficial].
- `Shape`: Represents an object in the scene, providing intersection methods and material properties [S2, unofficial].
- `Material`: Contains diffuse coefficient, emissive status, and methods to compute reflectance [S2, unofficial].
- `LightSource`: Represents a light emitter in the scene [S2, unofficial].

**Steps:**
1. **Intersection:** Find the closest intersection between the input ray and any object in `objList`. If no intersection is found, return the background color [S2, unofficial].
2. **Emissive Check:** If the intersected object's material is emissive, return its emissive radiance [S2, unofficial].
3. **Ambient Contribution:** Initialize the color to zero. Add an ambient term calculated as `ambient_constant * diffuse_coefficient`. The default `ambient_constant` is $(0.25, 0.25, 0.25)$ [S2, unofficial].
4. **Direct Illumination:** For each light source in `LightSourceList`:
    a. Calculate the vector from the intersection point to the light source.
    b. Compute the reflectance using the material's `getReflectance` method, passing the surface normal, view direction, and light direction [S2, unofficial].
    c. Cast a shadow ray from the intersection point to the light source. Set the shadow ray's `maxT` to the distance to the light source.
    d. If the shadow ray does not intersect any object (i.e., the light is visible), add the reflected radiance to the total color.
5. **Glossy Reflection (Optional):** If the material has a glossy component, compute the reflection direction $\vec{R}$ based on the surface normal and view direction. Distribute outgoing rays around $\vec{R}$ using a shininess exponent $\alpha$ to model rough surface reflections [S1, unofficial].

**Formulas:**
- **Lambertian Diffuse Reflection:** The reflected radiance $L_r(\vec{\omega}_r)$ is given by:
  $$L_r(\vec{\omega}_r) = \frac{\rho}{\pi} \int_{\Omega_i} L_i(\vec{\omega}_i) \cos \theta_i d\omega_i$$
  where $\rho$ is the material reflectance fraction, $L_i(\vec{\omega}_i)$ is the incident radiance, $\theta_i$ is the angle between the incident direction and the surface normal, and the integral is over the hemisphere $\Omega_i$ [S1, unofficial]. The BRDF $f_r$ is constant and related to $\rho$ by $\rho = \pi f_r$ [S1, unofficial].
- **Glossy Reflection:** The contribution is approximated by:
  $$L_r(\vec{\omega}_r) = L_r d\omega_r (\max(\vec{R} \cdot \vec{\omega}_r), 0)^\alpha$$
  where $\vec{R}$ is the perfect mirror reflection vector, $\vec{\omega}_r$ is the view direction, and $\alpha$ is the shininess exponent [S1, unofficial].

**Parameters:**
- `ambient_constant`: Default $(0.25, 0.25, 0.25)$ [S2, unofficial].
- $\alpha$ (shininess): Controls the narrowness of glossy highlights; higher values yield sharper highlights [S1, unofficial].
- `Epsilon`: $10^{-4}$, used to offset shadow ray origins to prevent self-intersection [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Self-Shadowing:** Shadow rays must be offset from the surface point by a small epsilon to avoid intersecting the surface they originated from [S2, unofficial].
- **Negative Dot Products:** In glossy reflection, dot products between $\vec{R}$ and $\vec{\omega}_r$ must be clamped to $[0, \infty)$ to handle cases where the reflection direction is behind the surface [S1, unofficial].
- **Emissive Objects:** Emissive objects should return their radiance directly without further shading calculations [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def whitted_shading(ray, scene):
    # 1. Find closest intersection
    hit = get_closest_intersection(ray, scene.objectsList)
    
    # 2. No intersection: return background
    if hit is None:
        return scene.background_color
    
    # 3. Emissive material: return emissive radiance
    if hit.material.is_emissive:
        return hit.material.emissive_radiance
    
    # 4. Initialize color with ambient contribution
    color = Vector3D(0.0, 0.0, 0.0)
    ambient_constant = Vector3D(0.25, 0.25, 0.25)
    color += ambient_constant * hit.material.diffuse_coefficient
    
    # 5. Direct illumination from lights
    for light in scene.lightSourceList:
        # Calculate vector to light
        light_dir = light.position - hit.point
        distance = light_dir.length()
        light_dir = light_dir.normalize()
        
        # Get reflectance
        reflectance = hit.material.get_reflectance(
            hit.normal, ray.direction, light_dir
        )
        
        # Cast shadow ray
        shadow_origin = hit.point + hit.normal * Epsilon
        shadow_ray = Ray(shadow_origin, light_dir, ray.depth + 1, Epsilon, distance)
        shadow_hit = get_closest_intersection(shadow_ray, scene.objectsList)
        
        # If light is visible, add contribution
        if shadow_hit is None:
            color += reflectance * light.intensity / (distance ** 2)
            
    return color
```

## Subtopics

### Whitted-Style Shading with Diffuse/Glossy Materials — Direct Illumination with Shadow Rays
A simplified variant that returns emissive radiance or diffuse reflectance based on material properties, using shadow rays to determine direct light visibility [S2, unofficial]. It finds the closest intersection, checks for emissive materials, and if no intersection is found, returns the background color [S2, unofficial].

### Glossy Reflection
Models reflections on rough surfaces by distributing outgoing rays around the perfect mirror reflection direction, controlled by a shininess exponent $\alpha$ [S1, unofficial]. It computes the dot product between the reflection vector $\vec{R}$ and view direction $\vec{\omega}_r$, clamps it to $[0, \infty)$, and raises it to the power of $\alpha$ to determine the intensity of the glossy highlight [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of light sources and objects in the scene for each ray, due to intersection tests and shadow ray casts [S2, unofficial].
- **Memory Complexity:** Linear with respect to the number of objects and lights stored in the scene [S2, unofficial].
- **Strengths:** Simple to implement; provides visually plausible results for scenes with few light sources; efficient for real-time rendering [S1, S2, unofficial].
- **Limitations:** Does not account for indirect lighting (global illumination) beyond direct shadows; glossy reflections are approximated and may not be physically accurate for complex rough surfaces [S1, S2, unofficial].
- **When to Use:** Suitable for real-time applications or scenes where direct lighting and shadows are the primary visual concerns [S2, unofficial].
- **When Not to Use:** Unsuitable for scenes requiring accurate global illumination, such as those with significant color bleeding or complex inter-reflections [S1, unofficial].

## Background
The concept of Lambertian diffuse reflection, where apparent brightness is independent of observer and light source positions, has been a fundamental model in computer graphics since the early days of rendering [S1, unofficial]. Whitted-style shading, named after Turner Whitted's 1980 paper "An Improved Illumination Model for Shaded Display," introduced recursive ray tracing for specular reflections and shadows [S1, unofficial]. The specific implementation with diffuse/glossy materials and ambient lighting as described here is commonly found in educational ray tracers and early real-time rendering systems [S2, unofficial].

## Sources
[S1] Real-Time Global Illumination Using OpenGL And Voxel Cone Tracing, https://doi.org/10.48550/arxiv.2104.00618, fetched 2026-10-26 (unofficial)
[S2] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
