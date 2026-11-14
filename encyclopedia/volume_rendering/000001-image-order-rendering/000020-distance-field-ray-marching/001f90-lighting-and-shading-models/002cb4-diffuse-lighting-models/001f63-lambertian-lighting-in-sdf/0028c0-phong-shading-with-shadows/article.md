# Phong Shading with Shadows

Also called: BRDF Lighting Model, Improved Lighting Models, Phong Model, Phong Relighting, Phong Shading Model, Standard Illumination Model

## Overview
Phong Shading with Shadows calculates the final color of a surface point in a volume rendering pipeline by combining ambient, diffuse, and specular lighting contributions using the Phong reflection model, modulated by a shadow factor [S4, unofficial]. It solves the problem of providing physically plausible surface appearance and depth cues for SDF-based volume rendering by accounting for occlusion from light sources [S4, unofficial]. Unlike its parent [Lambertian Lighting in SDF](2023-lambertian-lighting-in-sdf.md), which only computes diffuse lighting, this method adds specular highlights and ambient terms to better define surface curvature and material properties [S2, S5, unofficial]. It differs from [Sphere Tracing with Shading](3812-sphere-tracing-with-shading.md) by explicitly supporting multiple light types (positional, head, camera) and configurable attenuation, whereas sphere tracing variants often focus on the marching algorithm itself [S3, unofficial].

## How it works
**Inputs:**
- Surface position $p$ (world space).
- Surface normal $N$ (unit vector).
- Eye direction $V$ (unit vector, from $p$ to camera).
- Light direction $L$ (unit vector, from $p$ to light source).
- Material properties: ambient coefficient $k_a$, diffuse coefficient $k_d$, specular coefficient $k_s$, shininess exponent $k_p$ (or $n$) [S2, S4, unofficial].
- Light properties: color/intensity $I$, shadow factor $sh$ [S4, unofficial].

**Outputs:**
- Final color $C$ (RGB vector).

**Data Structures:**
The material properties are typically packed into a structure:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
struct Material:
    vec3 ka  # Ambient coefficient
    float kd # Diffuse coefficient
    float ks # Specular coefficient
    float kp # Shininess exponent
```
Light properties are packed as:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
struct Light:
    vec3 c   # Color/Intensity
    float intensity # Scalar intensity multiplier
```

**Steps:**

1. **Surface Normal Recovery:**
   Compute the surface normal $N$ from the Signed Distance Field (SDF) gradient $\nabla SDF(p)$ [S2, unofficial]:
   $$ N = \frac{\nabla SDF(p)}{\|\nabla SDF(p)\|} $$
   [G] Guard against zero-length gradients (e.g., in homogeneous regions) by returning a default normal or zeroing out lighting contributions to avoid artifacts [S2, G, unofficial].

2. **Normal Orientation:**
   Ensure the normal faces the viewer. If the dot product of the normal and the inverted ray direction (or eye direction) is negative, flip the normal [S1, S5, unofficial]:
   $$ \text{if } \dot(N, -V) < 0.0 \text{ then } N = -N $$
   Alternatively, using a step function:
   $$ N = N \times (\text{step}(0.0, \dot(N, V)) \times 2.0 - 1.0) $$
   [S5, unofficial]

3. **Light Direction Calculation:**
   For a positional light at $C_{light}$, compute the normalized light direction:
   $$ L = \frac{C_{light} - p}{\|C_{light} - p\|} $$
   [S4, unofficial]

4. **Shadow Factor Calculation:**
   Compute the shadow factor $sh$ by tracing a ray from $p$ towards the light source. This can be a hard shadow (binary 0 or 1) or soft shadow (continuous value based on distance to occluder) [S4, unofficial].
   [G] In SDF contexts, this is often done by checking if the SDF value along the ray to the light is less than a small epsilon, or by integrating the SDF along the path [S4, G, unofficial].

5. **Diffuse Lighting:**
   Calculate the diffuse term $C_{diff}$:
   $$ C_{diff} = k_d I \max(\dot(N, L), 0) $$
   [S4, unofficial]
   Implementation:
   ```python
   ndotl = max(dot(N, L), 0.0)
   diffuse = ndotl * col # where col is the base color modulated by kd and I
   ```
   [S5, unofficial]

6. **Specular Lighting:**
   Calculate the reflection vector $R$:
   $$ R = \text{reflect}(-L, N) $$
   [S1, unofficial]
   Calculate the specular term $C_{spec}$:
   $$ C_{spec} = k_s I \max(\dot(R, V), 0)^{k_p} $$
   [S4, unofficial]
   Implementation:
   ```python
   R = reflect(-L, N)
   rdotv = max(dot(R, V), 0.0)
   specular = pow(rdotv, kp) * ks * I
   ```
   [S5, unofficial]

7. **Ambient Lighting:**
   Calculate the ambient term:
   $$ C_{amb} = k_a I $$
   [S2, S5, unofficial]
   Implementation:
   ```python
   ambient = ka * I
   ```
   [S5, unofficial]

8. **Final Color Combination:**
   Combine the terms and apply the shadow factor:
   $$ C = (C_{amb} + C_{diff} + C_{spec}) \times sh $$
   [S4, unofficial]
   Clamp the result to $[0, 1]$ [S5, unofficial]:
   ```python
   result = ambient + diffuse + specular
   final_color = clamp(result, 0.0, 1.0)
   ```
   [S5, unofficial]

**Parameters:**
- $k_a$ (Ambient constant): Not given in the sources. [S2, unofficial]
- $k_d$ (Diffuse constant): Not given in the sources. [S2, unofficial]
- $k_s$ (Specular constant): Not given in the sources. [S2, unofficial]
- $k_p$ or $n$ (Shininess): Higher values make the surface smoother and shinier. Typical values range from 1 to 128, with 32 or 10 often used in examples [S2, S5, S1, unofficial].
- Ambient lighting factor: Constant 0.1 in some implementations [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Zero Gradient:** In homogeneous regions, the SDF gradient is zero, making normal recovery unstable. This leads to noise or incorrect shading [S6].
- **Denormalization Artifacts:** When interpolating normals or reflection vectors across pixels, denormalization can cause artifacts. Cube map dependent texture reads can avoid this but are slower [S6].
- **Shadow Artifacts:** Hard shadows can cause aliasing at shadow boundaries. Soft shadows require additional sampling or SDF queries [S4, unofficial].

## Subtopics

### [Lighting Transfer Function](6468-lighting-transfer-function.md)
The Lighting Transfer Function (LTF) is a volume rendering shading technique that replaces scalar lighting coefficients with lookup tables dependent on two scalar values sampled along the gradient direction, allowing users to control lighting specifically for material boundaries. It solves the problem of assigning specific lighting coefficients (ambient, diffuse, specular) to material boundaries and homogeneous regions in a volume, allowing users to control how light interacts with scalar data boundaries independently of opacity. Unlike methods using scalar gradient magnitude alone, this method uses scalar values perpendicular to the boundary (along the gradient) to distinguish material types and boundaries, avoiding the shading of homogeneous material using only scalar gradient magnitude.

### [Differentiable Blinn-Phong Light Estimation](8621-differentiable-blinn-phong-light-estimation.md)
This method solves the inverse problem of estimating the position and intensity of point light sources given an observed image, known scene geometry, and material properties. It formulates light estimation as an optimization problem that minimizes the photometric error between a rendered image and the target image using a differentiable rendering pipeline based on the Blinn-Phong model. It distinguishes itself from sibling methods like [Lighting Transfer Function](6468-lighting-transfer-function.md) by explicitly modeling 3D point light sources and handling non-distant lighting parallax, whereas alternatives like Spherical Harmonics are limited to low-frequency or distant lighting approximations.

### [Opacity Influenced Inconstant Method](14965-opacity-influenced-inconstant-method.md)
The Opacity Influenced Inconstant Method is a shading technique for 3D holographic pyramid rendering that extends Phong shading by introducing an opacity variable to enhance depth cues and reveal the internal structure of 3D models. It differs from its parent, [Phong Shading with Shadows](4356-phong-shading-with-shadows.md), by modulating diffuse and ambient lighting with opacity to create a transparent, holographic appearance, and by using a specific multi-light configuration to maximize shape perception without casting shadows. Unlike [Lighting Transfer Function](6468-lighting-transfer-function.md) which focuses on energy conservation, this method prioritizes visual depth cues for holographic displays.

### Cube Map Dependent Texture Read Shading
An efficient per-pixel surface shading method using cube map dependent texture reads to evaluate the Blinn-Phong model, avoiding the need for per-pixel vector normalization [S6].
**Steps:**
1. Treat an RGB color component (normal or reflection vector) as texture coordinates $(s, t, r)$ for a cube map [S6].
2. Access the texel at the intersection of the line through the origin in direction $(s, t, r)$ and the cube face [S6].
3. The color value represents incoming diffuse radiance if $(s, t, r)$ is a surface normal, or specular radiance if it is a reflection vector [S6].
**Parameters:**
- Performance Cost: Per-pixel cube map evaluation can be up to 3 times slower than evaluating dot products for a limited number of light sources in the fragment shader [S6].
**Pitfalls:**
- Denormalization artifacts: The original Rezk-Salama approach suffers from artifacts caused by denormalization during interpolation. Cube map reads avoid the need for renormalization [S6].
**Comparison:**
- Differs from standard fragment shader dot products by encoding arbitrary numbers of lights or a full environment map in the cube map, at the cost of performance [S6].

## Operating characteristics
- **Time Complexity:** $O(1)$ per pixel for lighting calculation, assuming shadow factor is precomputed or approximated. Shadow tracing adds $O(N)$ where $N$ is the number of samples along the light ray [S4, unofficial].
- **Memory Complexity:** $O(1)$ for material and light parameters. Cube map variant requires $O(1)$ texture memory for the cube map [S6].
- **Measured Performance:** Cube map dependent texture reads are up to 3 times slower than standard dot product evaluations for a limited number of lights [S6].
- **Strengths:** Provides realistic specular highlights and ambient lighting, enhancing depth perception and material definition. Supports multiple light types and attenuation [S3, unofficial].
- **Limitations:** Sensitive to noise in SDF gradients, leading to unstable normals in homogeneous regions. Shadow calculation can be expensive if not optimized [S6][S4, unofficial].
- **When to Use:** When realistic surface appearance with specular highlights is required, and the SDF is smooth enough to provide stable normals.
- **When Not to Use:** In highly noisy SDFs or homogeneous regions where normal estimation is unreliable, or when performance is critical and simple diffuse lighting suffices.

## Background
The Phong shading model was introduced by Bui Tuong Phong in 1973 [S3, G, unofficial]. It has been widely adopted in computer graphics for its simplicity and effectiveness in simulating specular highlights. In the context of volume rendering, it has been integrated into libraries like VTK.js to support cinematic volume rendering, providing BRDF reflectance and improved lighting models [S3, unofficial]. The integration with SDF-based rendering allows for dynamic relighting and shadow casting, enhancing the visual quality of 3D holographic and volume visualizations [S1, S2, unofficial].

## Sources
[S1] xwghua/clearvolume-plus, https://github.com/xwghua/clearvolume-plus, fetched 2026-10-26 (unofficial)
[S2] omkarchittar/Neural_Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
[S3] Cinematic Volume Rendering in Your Web Browser: VolView and VTK.js, https://www.kitware.com/cinematic-volume-rendering/, fetched 2026-10-27 (unofficial)
[S4] trimesh33/sdf, https://github.com/trimesh33/sdf, fetched 2026-10-27 (unofficial)
[S5] GitHub - mlavik1/UnityVolumeRendering: Volume rendering, implemented in ..., https://github.com/mlavik1/UnityVolumeRendering, fetched 2026-10-28 (unofficial)
[S6] Multidimensional Transfer Functions for Interactive Volume ..., https://www.academia.edu/5403340/Multidimensional_Transfer_Functions_for_Interactive_Volume_Rendering, fetched 2026-11-01
