# Phong Shading with Shadows

Also called: Lighting Calculation, Phong Relighting, Standard Illumination Model

## Overview
Phong Shading with Shadows calculates the final color of a surface point in a Signed Distance Field (SDF) by combining ambient, diffuse, and specular lighting components defined by the Phong reflection model, modulated by shadow factors [S2, unofficial][S3, unofficial]. It extends basic Lambertian shading by adding specular highlights and occlusion, allowing for the relighting of volume-rendered surfaces under directional lights [S1, unofficial]. Unlike [Sphere Tracing with Shading](3812-sphere-tracing-with-shading.md), which may use simpler shading approximations, this method explicitly computes the reflection vector for specular highlights and integrates shadow rays to determine visibility [S2, unofficial][S3, unofficial].

## How it works
**Inputs:**
- Surface position $p$ (from ray marching).
- Surface normal $n$ (recovered from SDF gradient).
- Light direction $l$ (normalized vector from $p$ to light source).
- Eye direction $v$ (normalized vector from $p$ to camera).
- Material properties: ambient coefficient $k_a$, diffuse coefficient $k_d$, specular coefficient $k_s$, shininess exponent $k_p$ (or $n$) [S1, unofficial][S2, unofficial].
- Light color/intensity $I$ [S2, unofficial].
- Shadow factor $sh$ (hard or soft) [S2, unofficial].

**Outputs:**
- Final color $C$ (clamped to $[0, 1]$) [S3, unofficial].

**Data Structures:**
The material and light properties are typically structured as follows [S2, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct material {
    vec3 ka; // Ambient color/intensity
    float kd; // Diffuse coefficient
    float ks; // Specular coefficient
    float kp; // Shininess exponent
};

struct light {
    vec3 c; // Light color
    float intensiv; // Light intensity
};
```

**Steps:**

1.  **Surface Normal Recovery**: Compute the gradient of the SDF at $p$, denoted $\nabla S(p)$, and normalize it to obtain the surface normal $n$ [S1, unofficial].
    $$ n = \frac{\nabla S(p)}{\|\nabla S(p)\|} $$
    [G] If the gradient norm is near zero (degenerate SDF), default to a standard normal or the view direction to avoid division by zero.

2.  **Normal Orientation**: Ensure the normal faces the viewer. If the dot product of the normal and the eye direction is negative, flip the normal [S3, unofficial].
    $$ n = n \times \text{step}(0.0, \text{dot}(n, v)) \times 2.0 - 1.0 $$
    [G] In practice, this is often implemented as: `if (dot(n, v) < 0.0) n = -n;`.

3.  **Light Direction**: Calculate the normalized vector from the surface point to the light source [S2, unofficial].
    $$ l = \text{normalize}(L_{pos} - p) $$
    For a directional light, $l$ is constant.

4.  **Shadow Factor**: Compute the shadow factor $sh$ by tracing a ray from $p$ in direction $l$. This can be a hard shadow (binary 0 or 1) or a soft shadow (based on distance to the nearest surface) [S2, unofficial].

5.  **Diffuse Lighting**: Calculate the diffuse component using the Lambertian reflectance model [S3, unofficial].
    $$ C_{diff} = k_d I \max(n \cdot l, 0) $$
    [S2, unofficial] defines this as $C_{diff} = k_d I \max(n \cdot l, 0)$.

6.  **Ambient Lighting**: Add the ambient component to simulate indirect light [S3, unofficial].
    $$ C_{amb} = k_a I $$
    [S3, unofficial] uses a constant `AMBIENT_LIGHTING_FACTOR` multiplied by the base color.

7.  **Specular Lighting**: Calculate the specular highlight using the Phong reflection model [S3, unofficial].
    - Compute the reflection vector $r$ of the light direction about the normal:
      $$ r = \text{reflect}(-l, n) $$
    - Compute the dot product between the reflection vector and the eye direction:
      $$ rdotv = \max(r \cdot v, 0) $$
    - Calculate the specular intensity:
      $$ C_{spec} = k_s I \cdot rdotv^{k_p} $$
    [S3, unofficial] uses a shininess of 32.0 and full white specular color.

8.  **Combine and Clamp**: Sum the components and clamp the result to the valid range [S3, unofficial].
    $$ C = (C_{amb} + C_{diff} + C_{spec}) \times sh $$
    $$ C_{final} = \min(C, 1.0) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def phong_shading_with_shadows(p, normal, eye_dir, light_pos, light_color, material, shadow_factor):
    # 1. Normalize inputs
    n = normalize(normal)
    v = normalize(eye_dir)
    l = normalize(light_pos - p)
    
    # 2. Flip normal if facing away from view
    if dot(n, v) < 0.0:
        n = -n
        
    # 3. Calculate Diffuse
    ndotl = max(dot(n, l), 0.0)
    diffuse = material.kd * light_color * ndotl
    
    # 4. Calculate Ambient
    ambient = material.ka * light_color
    
    # 5. Calculate Specular
    r = reflect(-l, n)
    rdotv = max(dot(r, v), 0.0)
    specular = material.ks * light_color * pow(rdotv, material.kp)
    
    # 6. Combine
    result = ambient + diffuse + specular
    result = result * shadow_factor
    
    # 7. Clamp
    return min(result, 1.0)
```

## Subtopics

### [Lighting Transfer Function](6468-lighting-transfer-function.md)
The Lighting Transfer Function (LTF) is a 2D lookup table technique that integrates multiple illumination models into a single, constant-cost evaluation for volume rendering. It solves the problem of efficiently applying complex illustrative shading (such as contour enhancement, cartoon shading, or metal shading) by storing ambient, diffuse, specular, and opacity enhancement components indexed by the dot products of the normalized gradient with the light and half-way vectors. Unlike standard Phong shading which calculates lighting per sample, the LTF decouples the shading model from the ray marching loop, allowing different visual styles to be swapped without changing the rendering algorithm.

### [Differentiable Blinn-Phong Light Estimation](8621-differentiable-blinn-phong-light-estimation.md)
This method solves the inverse problem of estimating the position and intensity of point light sources given an observed image, known scene geometry, and material properties. It formulates light estimation as an optimization problem that minimizes the photometric error between a rendered image and the target image using a differentiable rendering pipeline based on the Blinn-Phong model. It distinguishes itself from sibling methods like [Lighting Transfer Function](6468-lighting-transfer-function.md) by explicitly modeling 3D point light sources and handling non-distant lighting parallax, whereas alternatives like Spherical Harmonics are limited to low-frequency or distant lighting approximations.

## Operating characteristics
- **Time Complexity**: $O(N)$ per pixel, where $N$ is the number of ray marching steps. The lighting calculation is $O(1)$ per sample. Shadow rays add additional cost proportional to the shadow ray length [S2, unofficial].
- **Memory Complexity**: $O(1)$ for lighting parameters. SDF storage is external to this method.
- **Strengths**: Provides realistic specular highlights and soft/hard shadows, enhancing the perception of depth and material properties [S3, unofficial]. Simple to implement [S1, unofficial].
- **Limitations**: Computationally more expensive than basic diffuse shading due to specular calculation and shadow rays. The Phong model is an approximation and may not accurately represent real-world BRDFs [S1, unofficial].
- **When to use**: When visual quality with specular highlights and shadows is required for SDF-based volume rendering [S1, unofficial].
- **When not to use**: When performance is critical and simple shading suffices, or when physically based rendering is required (use PBR models instead).

## Background
The Phong shading model was introduced by Bui Tuong Phong in 1975 [S1, unofficial]. It is a widely used empirical model for calculating the intensity of light reflected from a surface. In the context of SDF volume rendering, it is often implemented as a post-processing or per-sample shading step to enhance the visual appearance of the rendered geometry [S1, unofficial][S2, unofficial]. The integration with shadows allows for more realistic scene composition [S2, unofficial].

## Sources
[S1] omkarchittar/Neural_Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
[S2] trimesh33/sdf, https://github.com/trimesh33/sdf, fetched 2026-10-27 (unofficial)
[S3] GitHub - mlavik1/UnityVolumeRendering: Volume rendering, implemented in ..., https://github.com/mlavik1/UnityVolumeRendering, fetched 2026-10-28 (unofficial)
