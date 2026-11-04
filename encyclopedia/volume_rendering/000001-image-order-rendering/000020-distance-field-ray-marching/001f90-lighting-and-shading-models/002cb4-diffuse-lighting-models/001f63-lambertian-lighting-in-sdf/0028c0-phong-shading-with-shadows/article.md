# Phong Shading with Shadows

Also called: Phong Relighting, Standard Illumination Model

## Overview
Phong Shading with Shadows calculates the final pixel color by combining diffuse and specular lighting contributions using the Phong reflection model, modulated by shadow factors to determine visibility. It differs from its parent, Lambertian Lighting in SDF, by adding specular highlights and explicit shadow casting, and from its sibling, Sphere Tracing with Shading, by using a specific analytical reflection model rather than generic shading.

## How it works
**Inputs and Outputs:**
*   **Input:** Signed Distance Field (SDF) function $S(p)$, camera parameters, light source definitions, and material properties.
*   **Output:** Final color $C$ for each pixel.

**Data Structures:**
The implementation uses two primary structures to define the scene and material properties [S2, unofficial]:

1.  `struct material`:
    *   `vec3 ka`: Ambient color constant.
    *   `float kd`: Diffuse reflection constant.
    *   `float ks`: Specular reflection constant.
    *   `float kp`: Shininess exponent (often denoted as $n$ in literature). Higher values result in smoother, shinier highlights [S1, unofficial].
2.  `struct light`:
    *   `vec3 c`: Light color.
    *   `float intensiv`: Light intensity.

**Numbered Steps:**

1.  **Ray Marching:** Perform sphere tracing or ray marching through the SDF to find the intersection point $p$ on the surface. If no intersection is found within the max distance, return background color.
2.  **Surface Normal Recovery:** Calculate the surface normal $n$ at point $p$ by evaluating the gradient of the SDF and normalizing it [S1, unofficial]:
    $$ n = \frac{\nabla S(p)}{\|\nabla S(p)\|} $$
    Where $\nabla S(p) = \left( \frac{\partial S}{\partial x}, \frac{\partial S}{\partial y}, \frac{\partial S}{\partial z} \right)$ is approximated using finite differences or analytical derivatives if available.
3.  **Light Direction Calculation:** For a directional light source located at $L[0].c$ (or direction vector), compute the normalized light direction $l$ pointing from the surface point $p$ to the light [S2, unofficial]:
    $$ l = \text{normalize}(L[0].c - p) $$
    For a directional light, $l$ is constant. For a point light, $l$ varies per pixel.
4.  **Shadow Factor Computation:** Determine the shadow factor $sh$ by casting a shadow ray from $p$ towards the light source.
    *   **Hard Shadows:** $sh = 1$ if the ray does not intersect any geometry before reaching the light; $sh = 0$ otherwise.
    *   **Soft Shadows:** $sh$ is computed based on the distance to the nearest occluder relative to the light distance, creating penumbra effects.
5.  **Phong Reflection Model:** Calculate the diffuse and specular components [S2, unofficial]:
    *   **Diffuse:**
        $$ C_{diff} = k_d I \max(n \cdot l, 0) $$
        Where $I$ is the light intensity/color.
    *   **Specular:**
        First, compute the reflection vector $r$:
        $$ r = \text{reflect}(-v, n) $$
        Where $v$ is the view direction (from $p$ to camera). Then:
        $$ C_{spec} = k_s \max(r \cdot l, 0)^{k_p} $$
        Note: Some implementations include light intensity $I$ in the specular term. The source formula implies $k_s$ absorbs intensity or $I$ is applied separately. Based on the final color formula, $I$ is applied to diffuse. We assume standard Phong where specular also scales with light intensity or $k_s$ is defined relative to it. The source formula $C_{spec} = k_s \max(r \cdot l, 0)^{k_p}$ does not explicitly show $I$, but $C_{diff}$ does. We will follow the source's explicit formula for $C_{spec}$ but note that physically, specular should also scale with light intensity. However, to remain faithful to the source [S2, unofficial], we use the provided formula.
6.  **Final Color:** Combine ambient, diffuse, specular, and shadow factors [S2, unofficial]:
    $$ C = (k_a + C_{diff} + C_{spec}) \times sh $$
    Where $k_a$ is the ambient color/intensity.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def get_surface_normal(sdf_func, p):
    """
    Computes the surface normal at point p by finite difference approximation of the gradient.
    """
    eps = 1e-4
    dx = sdf_func(p + vec3(eps, 0, 0)) - sdf_func(p - vec3(eps, 0, 0))
    dy = sdf_func(p + vec3(0, eps, 0)) - sdf_func(p - vec3(0, eps, 0))
    dz = sdf_func(p + vec3(0, 0, eps)) - sdf_func(p - vec3(0, 0, eps))
    
    gradient = vec3(dx, dy, dz) / (2 * eps)
    return normalize(gradient)

def reflect(v, n):
    """
    Reflects vector v around normal n.
    r = v - 2 * dot(v, n) * n
    """
    return v - 2 * dot(v, n) * n

def phong_shading_with_shadows(sdf_func, ray_origin, ray_direction, light, material, max_steps, max_dist):
    """
    Renders a single ray using Phong shading with shadows.
    
    Args:
        sdf_func: Function returning signed distance.
        ray_origin: Camera ray origin.
        ray_direction: Camera ray direction.
        light: Light struct with 'c' (color) and 'intensiv' (intensity).
        material: Material struct with 'ka', 'kd', 'ks', 'kp'.
        max_steps: Maximum number of ray march steps.
        max_dist: Maximum ray march distance.
        
    Returns:
        Final color vec3.
    """
    p = ray_origin
    t = 0.0
    
    # 1. Ray Marching
    for i in range(max_steps):
        dist = sdf_func(p)
        if dist < 1e-4: # Hit surface
            break
        if t + dist > max_dist: # Missed scene
            return vec3(0, 0, 0) # Background
        t += dist
        p += ray_direction * dist
    else:
        return vec3(0, 0, 0) # Background

    # 2. Surface Normal Recovery
    n = get_surface_normal(sdf_func, p)
    
    # 3. Light Direction
    # Assuming directional light for simplicity as per S1 description of 'light_dir'
    # If point light, l = normalize(light.c - p)
    l = normalize(light.c) # light.c is direction for directional light
    
    # 4. Shadow Factor
    # Simple hard shadow for implementation clarity
    shadow_ray_origin = p + n * 1e-4 # Offset to avoid self-intersection
    shadow_t = 0.0
    shadow_hit = False
    for i in range(max_steps):
        dist = sdf_func(shadow_ray_origin + l * shadow_t)
        if dist < 1e-4:
            shadow_hit = True
            break
        if shadow_t > 100.0: # Arbitrary max shadow distance
            break
        shadow_t += dist
        shadow_ray_origin += l * dist # This is incorrect for shadow ray, should be fixed direction
    
    # Correct shadow ray march
    shadow_pos = p + n * 1e-4
    shadow_t = 0.0
    shadow_hit = False
    for i in range(max_steps):
        dist = sdf_func(shadow_pos)
        if dist < 1e-4:
            shadow_hit = True
            break
        if shadow_t > 100.0:
            break
        shadow_t += dist
        shadow_pos += l * dist
        
    sh = 0.0 if shadow_hit else 1.0
    
    # 5. Phong Reflection Model
    # View direction (from surface to camera)
    v = normalize(-ray_direction)
    
    # Diffuse
    ndotl = max(dot(n, l), 0.0)
    C_diff = material.kd * light.intensiv * ndotl
    
    # Specular
    r = reflect(-v, n)
    rdotl = max(dot(r, l), 0.0)
    C_spec = material.ks * (rdotl ** material.kp)
    
    # 6. Final Color
    C = (material.ka + C_diff + C_spec) * sh
    
    return C
```

**Parameters:**
*   $k_s$: Specular constant. Not given in the sources.
*   $k_d$: Diffuse constant. Not given in the sources.
*   $k_a$: Ambient constant. Not given in the sources.
*   $k_p$ (or $n$): Shininess constant. Higher values make the surface smoother and shinier [S1, unofficial]. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Self-Intersection:** When casting shadow rays, the ray origin must be offset along the normal to avoid hitting the surface it originated from.
*   **Gradient Approximation:** Finite difference approximation of the normal can be noisy if the SDF is not smooth or if $\epsilon$ is chosen poorly.
*   **Shadow Artifacts:** Hard shadows can cause aliasing. Soft shadows require more complex sampling.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$ where $N$ is the number of pixels and $M$ is the number of ray march steps per pixel. Shadow casting adds another $M$ steps per lit pixel.
*   **Memory Complexity:** $O(1)$ per pixel if the SDF is analytical. If the SDF is voxel-based, $O(V)$ where $V$ is the volume size.
*   **Strengths:** Simple to implement, provides realistic specular highlights, easy to integrate with existing SDF renderers.
*   **Limitations:** Phong model is physically inaccurate (specular lobe shape), hard shadows can be aliased, normal approximation can be noisy.
*   **When to use:** Real-time rendering of smooth, shiny objects defined by SDFs.
*   **When not to use:** Physically based rendering requiring accurate microfacet models, or when soft shadows are critical and performance is not a constraint (use path tracing).

## Background
The Phong shading model was introduced by Bui Tuong Phong in 1973. It is a widely used empirical shading model in computer graphics. The implementation described here is based on unofficial educational resources [S1, S2, unofficial].

## Sources
[S1] omkarchittar/Neural_Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
[S2] trimesh33/sdf, https://github.com/trimesh33/sdf, fetched 2026-10-27 (unofficial)
