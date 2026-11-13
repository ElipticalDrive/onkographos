# Phong Shading with Shadows

Also called: BRDF Lighting Model, Improved Lighting Models, Lighting Calculation, Phong Model, Phong Relighting, Phong Shading Model, Standard Illumination Model

## Overview

Phong Shading with Shadows is a surface-based illumination model applied to volume rendering that computes diffuse, specular, and ambient lighting contributions using the Phong reflection model, modulated by shadow factors to simulate occlusion [S4, unofficial]. It solves the problem of adding realistic depth and material appearance to volume renderings by approximating surface normals from the scalar field's gradient and calculating light interaction per sample or per pixel [S3, unofficial][S5, unofficial]. Unlike the parent Lambertian model which only computes diffuse lighting, this method adds specular highlights and ambient terms, and unlike sibling Sphere Tracing with Shading which may use simplified shading, this method explicitly calculates shadow factors via secondary ray tracing to handle self-occlusion and external occlusion [S3, unofficial][S4, unofficial].

## How it works

### Inputs and Outputs
*   **Inputs**:
    *   Signed Distance Field (SDF) or scalar volume data.
    *   Camera position and ray direction for the current sample/pixel.
    *   Light source parameters (position, color, intensity, type).
    *   Material properties (ambient, diffuse, specular coefficients, shininess).
*   **Outputs**:
    *   Final color value (RGB) for the sample or pixel, incorporating lighting and shadow.

### Data Structures

**Material Structure** [S4, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```text
struct material {
    vec3 ka;      // Ambient color coefficient
    float kd;     // Diffuse coefficient
    float ks;     // Specular coefficient
    float kp;     // Shininess exponent
}
```

**Light Structure** [S4, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```text
struct light {
    vec3 c;       // Light color
    float intensiv; // Light intensity
}
```

### Parameters
*   `ka` (Ambient constant): Not given in the sources.
*   `kd` (Diffuse constant): Not given in the sources.
*   `ks` (Specular constant): Not given in the sources.
*   `kp` (Shininess constant): Higher values make the surface smoother and shinier [S2, unofficial].
*   Ambient term constant: 0.1 [S1, unofficial].
*   Diffuse weight: 0.6 [S1, unofficial].
*   Specular weight: 0.3 [S1, unofficial].
*   Specular exponent: 10.0 [S1, unofficial] or 32.0 [S5, unofficial].

### Steps

1.  **Surface Normal Recovery**: Compute the surface normal $N$ by calculating the gradient of the SDF at the current point $p$ and normalizing it [S2, unofficial].
    $$ N = \frac{\nabla SDF(p)}{\|\nabla SDF(p)\|} $$
    [G] If the gradient magnitude is zero (homogeneous region), the normal is undefined; typically, a default normal or zero vector is used, though sources note this is a limitation of gradient-based shading [S6].

2.  **Light Direction Calculation**: Calculate the vector from the current point $p$ to the light source position $L_{pos}$, then normalize it [S4, unofficial].
    $$ L = \text{normalize}(L_{pos} - p) $$
    [G] For directional lights, $L$ is constant. For positional lights, $L$ varies per sample.

3.  **Normal Orientation**: Ensure the normal faces the viewer. If the dot product of the normal and the view direction (or negative ray direction) is negative, flip the normal [S1, unofficial][S5, unofficial].
    ```python
    if dot(N, -rayDir) < 0.0:
        N = -N
    ```
    Alternatively, using a step function:
    ```python
    normal *= step(0.0, dot(normal, eyeDir)) * 2.0 - 1.0
    ```
    [S5, unofficial]

4.  **Shadow Factor Calculation**: Compute the shadow factor $sh$ by tracing a ray from the current point $p$ towards the light source. This can be a hard shadow (binary occlusion) or soft shadow (penumbra) [S4, unofficial].
    *   Hard Shadow: $sh = 1.0$ if the path to the light is clear, $0.0$ if occluded.
    *   Soft Shadow: $sh$ varies based on the distance to the occluder and the light size.
    [S3, unofficial] mentions secondary rays branching towards the light source to calculate global occlusion.

5.  **Diffuse Lighting**: Calculate the diffuse component $C_{diff}$ [S4, unofficial][S5, unofficial].
    $$ C_{diff} = k_d I \max(N \cdot L, 0) $$
    Where $I$ is the light intensity/color.
    [S1, unofficial] uses a simplified weight: $0.6 \times \max(N \cdot L, 0)$.

6.  **Specular Lighting**: Calculate the specular component $C_{spec}$ [S4, unofficial][S5, unofficial].
    *   Compute the reflection vector $R$:
        $$ R = \text{reflect}(-L, N) $$
    *   Compute the view direction $V$ (from point to camera).
    *   Calculate the specular term:
        $$ C_{spec} = k_s I \max(R \cdot V, 0)^{k_p} $$
    [S1, unofficial] uses exponent 10.0. [S5, unofficial] uses exponent 32.0 and full white specular color.

7.  **Ambient Lighting**: Calculate the ambient component $C_{amb}$ [S5, unofficial].
    $$ C_{amb} = k_a I_{ambient} $$
    [S1, unofficial] uses a constant factor 0.1.

8.  **Final Color Combination**: Combine the components and apply the shadow factor [S4, unofficial].
    $$ C = (C_{amb} + C_{diff} + C_{spec}) \times sh $$
    [S1, unofficial] combines them as: $0.1 + 0.6 \times diff + 0.3 \times spec$.
    [S5, unofficial] clamps the result to $[0, 1]$.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def phong_shading_with_shadows(p, ray_dir, light_pos, light_color, light_intensity, material, sdf_func):
    """
    Computes Phong shading with shadows for a volume rendering sample.
    
    Args:
        p: Current 3D point in space.
        ray_dir: Direction of the viewing ray (from camera to scene).
        light_pos: Position of the light source.
        light_color: Color of the light (RGB).
        light_intensity: Intensity of the light.
        material: Object containing ka, kd, ks, kp.
        sdf_func: Function to evaluate the Signed Distance Field.
        
    Returns:
        Final color (RGB) for the sample.
    """
    
    # 1. Surface Normal Recovery
    # Compute gradient using finite differences or analytical derivative
    eps = 1e-4
    grad_x = sdf_func(p + vec3(eps, 0, 0)) - sdf_func(p - vec3(eps, 0, 0))
    grad_y = sdf_func(p + vec3(0, eps, 0)) - sdf_func(p - vec3(0, eps, 0))
    grad_z = sdf_func(p + vec3(0, 0, eps)) - sdf_func(p - vec3(0, 0, eps))
    grad = vec3(grad_x, grad_y, grad_z)
    
    grad_norm = length(grad)
    if grad_norm < 1e-6:
        # Degenerate case: homogeneous region or inside object where gradient is zero
        # [G] Return black or handle based on context; sources note this is a limitation [S6]
        return vec3(0.0, 0.0, 0.0)
    
    N = normalize(grad)
    
    # 3. Normal Orientation (Flip if facing away from viewer)
    # View direction is opposite to ray direction
    view_dir = normalize(-ray_dir)
    if dot(N, view_dir) < 0.0:
        N = -N
        
    # 2. Light Direction
    L = normalize(light_pos - p)
    
    # 4. Shadow Factor Calculation
    # Trace ray from p towards light to check for occlusion
    # This is a simplified hard shadow check
    shadow_factor = 1.0
    # [G] Implement sphere tracing or ray marching towards light_pos
    # Check if any surface is closer to p than light_pos along direction L
    t = 0.0
    step_size = 0.01
    while t < length(light_pos - p):
        sample_point = p + L * t
        dist = sdf_func(sample_point)
        if dist < 0.0: # Inside an object
            shadow_factor = 0.0
            break
        t += step_size
        
    # 5. Diffuse Lighting
    ndotl = max(dot(N, L), 0.0)
    diffuse = material.kd * light_color * light_intensity * ndotl
    
    # 6. Specular Lighting
    # Reflection vector R = reflect(-L, N)
    # In GLSL: reflect(I, N) where I is incident vector (-L)
    R = reflect(-L, N)
    rdotv = max(dot(R, view_dir), 0.0)
    specular = material.ks * light_color * light_intensity * pow(rdotv, material.kp)
    
    # 7. Ambient Lighting
    ambient = material.ka * light_color * light_intensity # Assuming ambient light color is same as light or separate
    
    # 8. Final Color
    result = ambient + diffuse + specular
    result *= shadow_factor
    
    # Clamp to [0, 1]
    result = clamp(result, 0.0, 1.0)
    
    return result
```

## Subtopics

### [Lighting Transfer Function](6468-lighting-transfer-function.md)
The Lighting Transfer Function (LTF) is a 2D lookup table technique that integrates multiple illumination models into a single, constant-cost evaluation for volume rendering. It solves the problem of efficiently applying complex illustrative shading (such as contour enhancement, cartoon shading, or metal shading) by storing ambient, diffuse, specular, and opacity enhancement components indexed by the dot products of the normalized gradient with the light and half-way vectors. Unlike standard Phong shading which calculates lighting per sample, the LTF decouples the shading model from the ray marching loop, allowing different visual styles to be swapped without changing the rendering algorithm.

### [Differentiable Blinn-Phong Light Estimation](8621-differentiable-blinn-phong-light-estimation.md)
This method solves the inverse problem of estimating the position and intensity of point light sources given an observed image, known scene geometry, and material properties. It formulates light estimation as an optimization problem that minimizes the photometric error between a rendered image and the target image using a differentiable rendering pipeline based on the Blinn-Phong model. It distinguishes itself from sibling methods like [Lighting Transfer Function](6468-lighting-transfer-function.md) by explicitly modeling 3D point light sources and handling non-distant lighting parallax, whereas alternatives like Spherical Harmonics are limited to low-frequency or distant lighting approximations.

### Cube Map Dependent Texture Read Shading
An efficient per-pixel surface shading method using cube map dependent texture reads to evaluate the Blinn-Phong model, avoiding the need for per-pixel vector normalization [S6]. It works by treating an RGB color component (normal or reflection vector) as texture coordinates (s, t, r) for a cube map, accessing the texel at the intersection of the line through the origin in direction (s, t, r) and the cube face, where the color value represents incoming diffuse radiance if (s, t, r) is a surface normal, or specular radiance if it is a reflection vector [S6]. It differs from standard fragment shader dot products by encoding arbitrary numbers of lights or a full environment map in the cube map, at the cost of performance (up to 3 times slower) [S6]. A pitfall of the original Rezk-Salama approach is denormalization artifacts during interpolation, which cube map reads avoid by not requiring renormalization [S6].

## Operating characteristics

*   **Time Complexity**: O(1) per sample for lighting calculation, but requires secondary ray tracing for shadows, which adds significant cost depending on shadow depth and resolution [S3, unofficial][S4, unofficial].
*   **Memory Complexity**: O(1) for material and light parameters. LTF variant requires O(1) for the 2D lookup table [S1, unofficial][S2, unofficial].
*   **Measured Performance**: Per-pixel cube map evaluation can be up to 3 times slower than evaluating dot products for a limited number of light sources [S6].
*   **Strengths**: Provides realistic specular highlights and shadows, enhancing depth perception and material appearance [S3, unofficial][S4, unofficial]. Supports multiple light types (positional, head, camera) and properties (cone angle, attenuation, one-sided lighting) [S3, unofficial].
*   **Limitations**: Gradient-based normal estimation is sensitive to noise and fails in homogeneous regions (zero gradient) [S6]. Shadow calculation is expensive [S3, unofficial].
*   **When to Use**: When realistic lighting and shadows are required for volume visualization, and performance allows for secondary ray tracing [S3, unofficial][S4, unofficial].
*   **When Not to Use**: For real-time applications with strict performance constraints where shadows are not critical, or for homogeneous volumes where gradient-based shading fails [S6].

## Background

The Phong shading model was originally proposed by Bui Tuong Phong in 1973 [S1, unofficial]. It has been widely adopted in computer graphics for surface rendering. In the context of volume rendering, it has been integrated into libraries like VTK.js to support cinematic volume rendering, providing BRDF reflectance and improved lighting models [S3, unofficial]. The method is often implemented in educational and research contexts, such as neural surface rendering assignments [S2, unofficial], and in game engines like Unity for volume rendering [S5, unofficial]. The cube map dependent texture read variant was proposed by Rezk-Salama et al. to optimize per-pixel shading [S6].

## Sources

[S1] xwghua/clearvolume-plus, https://github.com/xwghua/clearvolume-plus, fetched 2026-10-26 (unofficial)
[S2] omkarchittar/Neural_Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
[S3] Cinematic Volume Rendering in Your Web Browser: VolView and VTK.js, https://www.kitware.com/cinematic-volume-rendering/, fetched 2026-10-27 (unofficial)
[S4] trimesh33/sdf, https://github.com/trimesh33/sdf, fetched 2026-10-27 (unofficial)
[S5] GitHub - mlavik1/UnityVolumeRendering: Volume rendering, implemented in ..., https://github.com/mlavik1/UnityVolumeRendering, fetched 2026-10-28 (unofficial)
[S6] Multidimensional Transfer Functions for Interactive Volume ..., https://www.academia.edu/5403340/Multidimensional_Transfer_Functions_for_Interactive_Volume_Rendering, fetched 2026-11-01
