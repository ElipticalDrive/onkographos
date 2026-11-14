# Phong Shading with Shadows

Also called: BRDF Lighting Model, Improved Lighting Models, Phong Model, Phong Relighting, Phong Shading Model, Standard Illumination Model

## Overview
Phong Shading with Shadows calculates the diffuse, specular, and ambient lighting contributions for a surface defined by a Signed Distance Field (SDF), modulated by shadow factors to simulate occlusion [S4, unofficial]. It extends basic Lambertian shading by adding specular highlights and shadowing, providing a more realistic appearance for volume boundaries [S3, unofficial]. Unlike its sibling [Sphere Tracing with Shading](3812-sphere-tracing-with-shading.md), which may use simpler shading or different normal approximations, this method explicitly implements the Phong reflection model with support for hard or soft shadows [S4, unofficial].

## How it works
**Inputs:**
- SDF function $S(p)$ returning distance at point $p$.
- Ray origin $O$ and direction $D$.
- Light position $L_{pos}$ (or direction $L_{dir}$ for directional lights).
- Material properties: ambient coefficient $k_a$, diffuse coefficient $k_d$, specular coefficient $k_s$, shininess exponent $k_p$ [S4, unofficial].
- Light color/intensity $I$ [S4, unofficial].
- Ambient lighting factor $k_{amb}$ [S5, unofficial].

**Outputs:**
- Final pixel color $C_{final}$ [S4, unofficial].

**Data Structures:**
The material properties are typically packed into a struct [S4, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct Material {
    vec3 ka; // Ambient color/intensity
    float kd; // Diffuse coefficient
    float ks; // Specular coefficient
    float kp; // Shininess exponent
};

struct Light {
    vec3 c; // Light color
    float intensiv; // Light intensity
};
```

**Steps:**

1. **Ray Marching:** Trace the ray through the SDF to find the intersection point $p$ and the number of steps. If no intersection is found within the maximum distance, return background color [S4, unofficial].

2. **Surface Normal Recovery:** Calculate the surface normal $N$ at $p$ by computing the gradient of the SDF and normalizing it [S2, unofficial]:
   $$ N = \frac{\nabla S(p)}{\|\nabla S(p)\|} $$
   [S2, unofficial]

3. **Light Direction:** Calculate the normalized light direction vector $L$ from the surface point $p$ to the light source [S4, unofficial]:
   $$ L = \text{normalize}(L_{pos} - p) $$
   [S4, unofficial]

4. **Shadow Calculation:** Compute a shadow factor $sh$ by casting a secondary ray from $p$ towards the light source. This can be a hard shadow (binary occlusion) or soft shadow (penumbra based on distance to occluder) [S4, unofficial].
   $$ sh = \text{ShadowFactor}(p, L) $$
   [S4, unofficial]

5. **Normal Orientation:** Ensure the normal faces the viewer. If the dot product of the normal and the view direction (negative ray direction) is negative, flip the normal [S1, S5, unofficial]:
   $$ \text{if } (\dot(N, -D) < 0.0) \text{ then } N = -N $$
   [S1, unofficial]
   Alternatively, using a step function:
   $$ N = N \times (\text{step}(0.0, \dot(N, \text{eyeDir})) \times 2.0 - 1.0) $$
   [S5, unofficial]

6. **Diffuse Lighting:** Calculate the diffuse component $C_{diff}$ using the Lambertian reflectance model [S4, unofficial]:
   $$ C_{diff} = k_d I \max(\dot(N, L), 0) $$
   [S4, unofficial]

7. **Specular Lighting:** Calculate the reflection vector $R$ and the specular component $C_{spec}$ [S4, unofficial]:
   $$ R = \text{reflect}(-L, N) $$
   [S1, unofficial]
   $$ C_{spec} = k_s I \max(\dot(R, -D), 0)^{k_p} $$
   [S4, unofficial]
   Note: Some implementations use the view vector $V$ instead of $-D$ if the view direction is distinct from the ray direction (e.g., in perspective vs orthographic or specific camera setups), but typically $V = -D$ for ray marching [S5, unofficial]. The specular intensity is scaled by $k_s$ and raised to the power of shininess $k_p$ [S4, unofficial].

8. **Ambient Lighting:** Calculate the ambient component $C_{amb}$ [S5, unofficial]:
   $$ C_{amb} = k_{amb} \times \text{MaterialColor} $$
   [S5, unofficial]
   Where $k_{amb}$ is a constant factor (e.g., 0.1) [S1, unofficial].

9. **Final Color:** Combine the components and apply the shadow factor [S4, unofficial]:
   $$ C = (C_{amb} + C_{diff} + C_{spec}) \times sh $$
   [S4, unofficial]
   Clamp the result to $[0, 1]$ [S5, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def phong_shading_with_shadows(p, normal, light_pos, ray_dir, material, light_color, ambient_factor, shadow_func):
    """
    Calculate Phong shading with shadows for a surface point.
    
    Args:
        p: Surface point position (vec3)
        normal: Surface normal (vec3), assumed normalized
        light_pos: Light position (vec3)
        ray_dir: Ray direction (vec3), assumed normalized
        material: Material struct with ka, kd, ks, kp
        light_color: Light color/intensity (vec3)
        ambient_factor: Ambient lighting factor (float)
        shadow_func: Function to calculate shadow factor (float)
        
    Returns:
        Final color (vec3)
    """
    # 1. Light direction
    L = normalize(light_pos - p)
    
    # 2. Shadow factor
    sh = shadow_func(p, light_pos)
    
    # 3. View direction (opposite of ray direction)
    V = -ray_dir
    
    # 4. Diffuse component
    ndotl = max(dot(normal, L), 0.0)
    diffuse = material.kd * light_color * ndotl
    
    # 5. Specular component
    R = reflect(-L, normal)
    rdotv = max(dot(R, V), 0.0)
    specular = material.ks * light_color * pow(rdotv, material.kp)
    
    # 6. Ambient component
    ambient = ambient_factor * material.ka
    
    # 7. Combine and apply shadow
    color = (ambient + diffuse + specular) * sh
    
    # 8. Clamp
    color = clamp(color, 0.0, 1.0)
    
    return color
```

**Parameters:**
- $k_a$ (ambient): Constant ambient lighting factor, typically 0.1 [S1, unofficial].
- $k_d$ (diffuse): Diffuse coefficient, typically 1.0 [S2, unofficial].
- $k_s$ (specular): Specular coefficient, typically 1.0 [S2, unofficial].
- $k_p$ (shininess): Shininess exponent, higher values make the surface smoother and shinier [S2, unofficial]. Typical values range from 10 to 100 [S1, S5, unofficial].
- Ambient factor: Constant multiplier for ambient light, e.g., 0.1 [S1, unofficial].

**Edge Cases:**
- **Degenerate Normals:** If the gradient magnitude is zero (homogeneous region), the normal is undefined. This method relies on surface normals, so it is not robust for shading homogeneous regions [S6].
- **Shadow Artifacts:** Hard shadows can cause aliasing at shadow boundaries. Soft shadows require additional sampling or approximation [S4, unofficial].
- **Precision:** Floating-point precision issues may arise in shadow ray marching, leading to self-shadowing artifacts (shadow acne). A small epsilon offset may be needed [G].

## Subtopics

### [Lighting Transfer Function](6468-lighting-transfer-function.md)
The Lighting Transfer Function (LTF) is a volume rendering shading technique that replaces scalar lighting coefficients with lookup tables dependent on two scalar values sampled along the gradient direction, allowing users to control lighting specifically for material boundaries. It solves the problem of assigning specific lighting coefficients (ambient, diffuse, specular) to material boundaries and homogeneous regions in a volume, allowing users to control how light interacts with scalar data boundaries independently of opacity. Unlike methods using scalar gradient magnitude alone, this method uses scalar values perpendicular to the boundary (along the gradient) to distinguish material types and boundaries, avoiding the shading of homogeneous material using only scalar gradient magnitude.

### [Differentiable Blinn-Phong Light Estimation](8621-differentiable-blinn-phong-light-estimation.md)
This method solves the inverse problem of estimating the position and intensity of point light sources given an observed image, known scene geometry, and material properties. It formulates light estimation as an optimization problem that minimizes the photometric error between a rendered image and the target image using a differentiable rendering pipeline based on the Blinn-Phong model. It distinguishes itself from sibling methods like [Lighting Transfer Function](6468-lighting-transfer-function.md) by explicitly modeling 3D point light sources and handling non-distant lighting parallax, whereas alternatives like Spherical Harmonics are limited to low-frequency or distant lighting approximations.

### Cube Map Dependent Texture Read Shading
An efficient per-pixel surface shading method using cube map dependent texture reads to evaluate the Blinn-Phong model, avoiding the need for per-pixel vector normalization [S6]. It treats an RGB color component (normal or reflection vector) as texture coordinates (s, t, r) for a cube map. The texel at the intersection of the line through the origin in direction (s, t, r) and the cube face is accessed. The color value represents incoming diffuse radiance if (s, t, r) is a surface normal, or specular radiance if it is a reflection vector [S6]. This approach encodes arbitrary numbers of lights or a full environment map in the cube map, at the cost of performance (up to 3 times slower than dot products for limited lights) [S6]. It avoids denormalization artifacts caused by interpolation in the original Rezk-Salama approach [S6].

## Operating characteristics
- **Time Complexity:** The shading calculation per sample is $O(1)$, but shadow calculation adds additional ray marching steps, making it dependent on the shadow algorithm (hard vs soft) [S4, unofficial].
- **Memory Complexity:** $O(1)$ for material and light parameters [S4, unofficial].
- **Performance:** Cube map dependent texture read shading can be up to 3 times slower than evaluating dot products for a limited number of light sources [S6].
- **Strengths:** Provides realistic specular highlights and shadows, enhancing the perception of shape and material [S3, unofficial]. Supports multiple light types (positional, head, camera) [S3, unofficial].
- **Limitations:** Not robust for shading homogeneous regions where the gradient is zero or noisy [S6]. Sensitive to normal estimation errors [S6].

## Background
The Phong shading model was introduced by Bui Tuong Phong in 1973 [S3, implied, unofficial]. It is a widely used local illumination model in computer graphics. In the context of volume rendering, it has been integrated into libraries like VTK.js to support cinematic volume rendering, providing BRDF reflectance calculations [S3, unofficial]. The method is commonly used in direct volume rendering to enhance the visual quality of scalar fields by simulating surface-like lighting on isosurfaces or gradient-based shading [S3, unofficial].

## Sources
[S1] xwghua/clearvolume-plus, https://github.com/xwghua/clearvolume-plus, fetched 2026-10-26 (unofficial)
[S2] omkarchittar/Neural_Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
[S3] Cinematic Volume Rendering in Your Web Browser: VolView and VTK.js, https://www.kitware.com/cinematic-volume-rendering/, fetched 2026-10-27 (unofficial)
[S4] trimesh33/sdf, https://github.com/trimesh33/sdf, fetched 2026-10-27 (unofficial)
[S5] GitHub - mlavik1/UnityVolumeRendering: Volume rendering, implemented in ..., https://github.com/mlavik1/UnityVolumeRendering, fetched 2026-10-28 (unofficial)
[S6] Multidimensional Transfer Functions for Interactive Volume ..., https://www.academia.edu/5403340/Multidimensional_Transfer_Functions_for_Interactive_Volume_Rendering, fetched 2026-11-01
