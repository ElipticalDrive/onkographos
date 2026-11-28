# Phong Shading with Shadows

Also called: BRDF Lighting Model, Improved Lighting Models, Phong Model, Phong Relighting, Phong Shading Model, Phong Shading for Surface Faces, Standard Illumination Model

## Overview
Phong Shading with Shadows calculates the final color of a surface point in a volume rendering pipeline by combining ambient, diffuse, and specular lighting contributions using the Phong reflection model, modulated by a shadow factor [S4, unofficial]. It solves the problem of providing physically plausible surface appearance and depth cues for SDF-based volume rendering by accounting for occlusion from light sources [S4, unofficial]. Unlike its parent [Lambertian Lighting in SDF](2023-lambertian-lighting-in-sdf.md), which only computes diffuse lighting, this method adds specular highlights and ambient terms to better define surface curvature and material properties [S2, S5, unofficial]. It differs from [Sphere Tracing with Shading](3812-sphere-tracing-with-shading.md) by explicitly supporting multiple light types (positional, head, camera) and configurable attenuation, whereas sphere tracing variants often focus on the marching algorithm itself [S3, unofficial].

## How it works
**Inputs:**
- Surface position $p$ (world space) [S5, unofficial].
- Surface normal $N$ (unit vector) [S2, S5, unofficial].
- Eye direction $V$ (unit vector, from $p$ to camera) [S5, unofficial].
- Light direction $L$ (unit vector, from $p$ to light source) [S4, S5, unofficial].
- Material properties: ambient coefficient $k_a$, diffuse coefficient $k_d$, specular coefficient $k_s$, shininess exponent $k_p$ (or $n$) [S3, S5, unofficial].
- Light properties: color/intensity $I$ [S5, unofficial].
- Shadow factor $sh$ [S4, unofficial].

**Outputs:**
- Final color $C$ (RGB vector) [S5, unofficial].

**Data Structures:**
The material properties are packed into a structure:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Material:
    ka: float  # Ambient coefficient
    kd: float  # Diffuse coefficient
    ks: float  # Specular coefficient
    kp: float  # Shininess exponent
```
Light properties are packed as:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Light:
    c: tuple[float, float, float]  # Color/Intensity
    intensity: float               # Scalar intensity multiplier
```

**Steps:**

1. **Surface Normal Recovery:**
   Compute the surface normal $N$ from the Signed Distance Field (SDF) gradient $\nabla SDF(p)$ [S3, unofficial]:
   $$ N = \frac{\nabla SDF(p)}{\|\nabla SDF(p)\|} $$
   [G] Guard against zero-length gradients in homogeneous regions by returning a default normal or zeroing out lighting contributions to avoid artifacts [S7].

2. **Normal Orientation:**
   Ensure the normal faces the viewer by flipping it if the dot product with the inverted ray direction is negative [S2, unofficial]:
   $$ \text{if } N \cdot (-V) < 0.0 \text{ then } N = -N $$
   Alternatively, using a step function:
   $$ N = N \times (\text{step}(0.0, N \cdot V) \times 2.0 - 1.0) $$
   [S6, unofficial]

3. **Light Direction Calculation:**
   For a positional light at $C_{light}$, compute the normalized light direction:
   $$ L = \frac{C_{light} - p}{\|C_{light} - p\|} $$
   [S5, unofficial]

4. **Shadow Factor Calculation:**
   Compute the shadow factor $sh$ by tracing a ray from $p$ towards the light source to determine global occlusion [S4, unofficial].
   [G] In SDF contexts, this is typically done by checking if the SDF value along the ray to the light is less than a small epsilon, or by integrating the SDF along the path [S4, G, unofficial].

5. **Diffuse Lighting:**
   Calculate the diffuse term $C_{diff}$:
   $$ C_{diff} = k_d I \max(N \cdot L, 0) $$
   [S5, unofficial]
   Implementation:
   ```python
   ndotl = max(N[0]*L[0] + N[1]*L[1] + N[2]*L[2], 0.0)
   diffuse = (material.kd * light.intensity * ndotl) * light.c
   ```
   [S5, unofficial]

6. **Specular Lighting:**
   Calculate the reflection vector $R$:
   $$ R = \text{reflect}(-L, N) $$
   [S1]
   Calculate the specular term $C_{spec}$:
   $$ C_{spec} = k_s I \max(R \cdot V, 0)^{k_p} $$
   [S5, unofficial]
   Implementation:
   ```python
   dot_N_negL = -(N[0]*L[0] + N[1]*L[1] + N[2]*L[2])
   R = [2 * dot_N_negL * n - l for n, l in zip(N, L)]
   rdotv = max(R[0]*V[0] + R[1]*V[1] + R[2]*V[2], 0.0)
   specular = (material.ks * light.intensity * (rdotv ** material.kp)) * light.c
   ```
   [S5, unofficial]

7. **Ambient Lighting:**
   Calculate the ambient term:
   $$ C_{amb} = k_a I $$
   [S5, unofficial]
   Implementation:
   ```python
   ambient = material.ka * light.intensity
   ```
   [S5, unofficial]

8. **Final Color Combination:**
   Combine the terms and apply the shadow factor:
   $$ C = (C_{amb} + C_{diff} + C_{spec}) \times sh $$
   [S5, unofficial]
   Clamp the result to $[0, 1]$ [S5, unofficial]:
   ```python
   result = [ambient + diffuse[i] + specular[i] for i in range(3)]
   final_color = [min(c, 1.0) for c in result]
   return final_color
   ```
   [S5, unofficial]

**Parameters:**
- $k_a$ (Ambient constant): Defined in the material structure [S5, unofficial].
- $k_d$ (Diffuse constant): Defined in the material structure [S5, unofficial].
- $k_s$ (Specular constant): Defined in the material structure [S5, unofficial].
- $k_p$ or $n$ (Shininess): Higher values make the surface smoother and shinier [S3, unofficial].
- Ambient lighting factor: Constant 0.1 in some implementations [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Zero Gradient:** In homogeneous regions, the SDF gradient is zero, making normal recovery unstable and leading to noise or incorrect shading [S7].
- **Denormalization Artifacts:** When interpolating normals or reflection vectors across pixels, denormalization can cause artifacts [S7].
- **Shadow Artifacts:** Hard shadows can cause aliasing at shadow boundaries [S4, unofficial].

## Subtopics

### [Lighting Transfer Function](6468-lighting-transfer-function.md)
The Lighting Transfer Function (LTF) is a volume rendering shading technique that replaces scalar lighting coefficients with lookup tables dependent on two scalar values sampled along the gradient direction, allowing users to control lighting specifically for material boundaries. It solves the problem of assigning specific lighting coefficients (ambient, diffuse, specular) to material boundaries and homogeneous regions in a volume, allowing users to control how light interacts with scalar data boundaries independently of opacity. Unlike methods using scalar gradient magnitude alone, this method uses scalar values perpendicular to the boundary (along the gradient) to distinguish material types and boundaries, avoiding the shading of homogeneous material using only scalar gradient magnitude.

### [Differentiable Blinn-Phong Light Estimation](8621-differentiable-blinn-phong-light-estimation.md)
This method solves the inverse problem of estimating the position and intensity of point light sources given an observed image, known scene geometry, and material properties. It formulates light estimation as an optimization problem that minimizes the photometric error between a rendered image and the target image using a differentiable rendering pipeline based on the Blinn-Phong model. It distinguishes itself from sibling methods like [Lighting Transfer Function](6468-lighting-transfer-function.md) and its parent [Phong Shading with Shadows](4356-phong-shading-with-shadows.md) by explicitly modeling 3D point light sources and handling non-distant lighting parallax, whereas alternatives like Spherical Harmonics are limited to low-frequency or distant lighting approximations.

### [Opacity Influenced Inconstant Method](14965-opacity-influenced-inconstant-method.md)
The Opacity Influenced Inconstant Method is a shading technique for 3D holographic pyramid rendering that extends Phong shading by introducing an opacity variable to enhance depth cues and reveal the internal structure of 3D models. It differs from its parent, [Phong Shading with Shadows](4356-phong-shading-with-shadows.md), by modulating diffuse and ambient lighting with opacity to create a transparent, holographic appearance, and by using a specific multi-light configuration to maximize shape perception without casting shadows. Unlike [Lighting Transfer Function](6468-lighting-transfer-function.md) which focuses on energy conservation, this method prioritizes visual depth cues for holographic displays.

### Cube Map Dependent Texture Read Shading
An efficient per-pixel surface shading method using cube map dependent texture reads to evaluate the Blinn-Phong model, avoiding the need for per-pixel vector normalization [S7].
**Steps:**
1. Treat an RGB color component (normal or reflection vector) as texture coordinates $(s, t, r)$ for a cube map [S7].
2. Access the texel at the intersection of the line through the origin in direction $(s, t, r)$ and the cube face [S7].
3. The color value represents incoming diffuse radiance if $(s, t, r)$ is a surface normal, or specular radiance if it is a reflection vector [S7].
**Parameters:**
- Performance Cost: Per-pixel cube map evaluation can be up to 3 times slower than evaluating dot products for a limited number of light sources in the fragment shader [S7].
**Pitfalls:**
- Denormalization artifacts: The original Rezk-Salama approach suffers from artifacts caused by denormalization during interpolation [S7].
**Comparison:**
- Differs from standard fragment shader dot products by encoding arbitrary numbers of lights or a full environment map in the cube map, at the cost of performance [S7].

## Operating characteristics
- **Time Complexity:** $O(1)$ per pixel for lighting calculation, assuming shadow factor is precomputed or approximated [S4, unofficial]. Shadow tracing adds $O(N)$ where $N$ is the number of samples along the light ray [S4, unofficial].
- **Memory Complexity:** $O(1)$ for material and light parameters [S5, unofficial]. Cube map variant requires $O(1)$ texture memory for the cube map [S7].
- **Measured Performance:** Cube map dependent texture reads are up to 3 times slower than standard dot product evaluations for a limited number of lights [S7].
- **Strengths:** Provides realistic specular highlights and ambient lighting, enhancing depth perception and material definition [S3, unofficial]. Supports multiple light types and attenuation [S4, unofficial].
- **Limitations:** Sensitive to noise in SDF gradients, leading to unstable normals in homogeneous regions [S7]. Shadow calculation can be expensive if not optimized [S4, unofficial].
- **When to Use:** When realistic surface appearance with specular highlights is required, and the SDF is smooth enough to provide stable normals [G].
- **When Not to Use:** In highly noisy SDFs or homogeneous regions where normal estimation is unreliable, or when performance is critical and simple diffuse lighting suffices [G].

## Background
The Phong shading model has been integrated into libraries like VTK.js to support cinematic volume rendering, providing BRDF reflectance and improved lighting models [S4, unofficial]. The integration with SDF-based rendering allows for dynamic relighting and shadow casting, enhancing the visual quality of 3D holographic and volume visualizations [S4, unofficial].

## Sources
[S1] Surface and volume rendering in three-dimensional imaging: A comparison, https://doi.org/10.1007/bf03168161, fetched 2026-10-26
[S2] xwghua/clearvolume-plus, https://github.com/xwghua/clearvolume-plus, fetched 2026-10-26 (unofficial)
[S3] omkarchittar/Neural_Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
[S4] Cinematic Volume Rendering in Your Web Browser: VolView and VTK.js, https://www.kitware.com/cinematic-volume-rendering/, fetched 2026-10-27 (unofficial)
[S5] trimesh33/sdf, https://github.com/trimesh33/sdf, fetched 2026-10-27 (unofficial)
[S6] GitHub - mlavik1/UnityVolumeRendering: Volume rendering, implemented in ..., https://github.com/mlavik1/UnityVolumeRendering, fetched 2026-10-28 (unofficial)
[S7] Multidimensional Transfer Functions for Interactive Volume ..., https://www.academia.edu/5403340/Multidimensional_Transfer_Functions_for_Interactive_Volume_Rendering, fetched 2026-11-01
