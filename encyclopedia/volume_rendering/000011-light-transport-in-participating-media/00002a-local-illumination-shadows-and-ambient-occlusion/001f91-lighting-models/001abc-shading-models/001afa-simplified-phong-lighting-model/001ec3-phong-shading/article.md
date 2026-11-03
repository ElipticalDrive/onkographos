# Phong Shading

Also called: Phong Illumination, Phong Illumination Model, Phong Lighting Model, Phong Reflection Model, Phong Shading Model, Phong model

## Overview
Phong Shading is a local illumination model that calculates the color of a surface point by summing ambient, diffuse, and specular components [S1, S2, S7]. It approximates direct lighting by combining a constant ambient term with a diffuse term proportional to the surface normal and light direction, and a specular term based on the alignment between the view direction and the surface reflection vector [S1, S2]. Unlike physically based models, it is an empirical approximation that does not account for occlusion, interreflection, or energy conservation [S2]. It is distinguished from its sibling [Blinn-Phong Shading Model](2239-blinn-phong-shading-model.md) by using the explicit reflection vector $R$ rather than the halfway vector $H$ for the specular calculation.

## How it works
**Inputs:**
*   Surface position $x$ (world space).
*   Surface normal $N$ (unit vector, world space).
*   View direction $V$ (unit vector, from $x$ to camera).
*   Light direction $L$ (unit vector, from $x$ to light source).
*   Material parameters: ambient coefficient $k_a$, diffuse coefficient $k_d$, specular coefficient $k_s$, shininess exponent $\alpha$ (or $n$).
*   Light parameters: ambient intensity $I_a$, diffuse intensity $I_{d}$, specular intensity $I_{s}$ (often represented as light colors).
*   Material colors: ambient color $O_a$, diffuse color $O_d$, specular color $O_s$ [S1].

**Outputs:**
*   Fragment color $C$ (RGB).

**Data Structures:**
*   **Material:** A structure containing floats for $k_a, k_d, k_s, \alpha$ and RGB vectors for $O_a, O_d, O_s$.
*   **Light:** A structure containing RGB vectors for ambient, diffuse, and specular intensities.

**Steps:**

1.  **Transform Vectors:** Ensure $N$, $L$, and $V$ are normalized unit vectors in the same coordinate space (typically world space or view space). If using a graphics pipeline, transform vertex positions and normals using the model matrix $M$ and view-projection matrices [S2].
    *   World position: $p_{world} = M \cdot p_{local}$ [S2].
    *   World normal: $n_{world} = (M^{-1})^T \cdot n_{local}$ (using the upper-left 3x3 submatrix of $M$) [S2].

2.  **Compute Reflection Vector:** Calculate the direction of perfect reflection $R$ for the light source.
    $$R = 2(N \cdot L)N - L$$
    where $N \cdot L$ is the dot product [S2, S4].

3.  **Compute Ambient Component:** Calculate the ambient contribution, which is constant regardless of geometry.
    $$C_{ambient} = k_a O_a I_a$$
    [S1, S7]. Note: Some implementations assume $O_a = O_d$ or simplify $I_a$ [S1].

4.  **Compute Diffuse Component:** Calculate the Lambertian reflection.
    $$C_{diffuse} = k_d O_d I_d \max(N \cdot L, 0)$$
    [S1, S7]. The $\max$ function ensures back-facing surfaces receive no diffuse light.

5.  **Compute Specular Component:** Calculate the specular highlight based on the angle between the reflection vector $R$ and the view vector $V$.
    $$C_{specular} = k_s O_s I_s \max(R \cdot V, 0)^\alpha$$
    [S1, S2]. Here $\alpha$ is the shininess exponent. Higher values produce smaller, sharper highlights.

6.  **Combine:** Sum the components.
    $$C = C_{ambient} + C_{diffuse} + C_{specular}$$
    [S2, S7].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def phong_shading(position, normal, view_direction, light_direction, material, light):
    """
    Calculate Phong shading for a single light source.
    
    Args:
        position: vec3, world space position of the fragment.
        normal: vec3, normalized world space surface normal.
        view_direction: vec3, normalized direction from position to camera.
        light_direction: vec3, normalized direction from position to light.
        material: dict or object with keys: ka, kd, ks, shininess, color_ambient, color_diffuse, color_specular.
        light: dict or object with keys: ambient, diffuse, specular (RGB vectors).
        
    Returns:
        vec3, final fragment color.
    """
    # 1. Normalize inputs (ensure unit vectors)
    N = normalize(normal)
    L = normalize(light_direction)
    V = normalize(view_direction)
    
    # 2. Ambient Component
    # C_ambient = k_a * O_a * I_a
    ambient = material['ka'] * material['color_ambient'] * light['ambient']
    
    # 3. Diffuse Component
    # C_diffuse = k_d * O_d * I_d * max(N . L, 0)
    diff_factor = max(dot(N, L), 0.0)
    diffuse = material['kd'] * material['color_diffuse'] * light['diffuse'] * diff_factor
    
    # 4. Specular Component
    # R = 2 * (N . L) * N - L
    # C_specular = k_s * O_s * I_s * max(R . V, 0)^shininess
    R = 2.0 * dot(N, L) * N - L
    spec_factor = max(dot(R, V), 0.0)
    specular = material['ks'] * material['color_specular'] * light['specular'] * (spec_factor ** material['shininess'])
    
    # 5. Combine
    color = ambient + diffuse + specular
    
    return color
```

**Parameters:**
*   **Shininess ($\alpha$ or $n$):** Controls the size of the specular highlight. Default values in implementations often range from 10 to 64 [S1, S4]. For a perfect reflector, the exponent is infinite [S1].
*   **Coefficients ($k_a, k_d, k_s$):** Typically in the range [0, 1]. Some implementations use values > 1 for specular to simulate energy gain (non-physical) [S1, S4].
*   **Colors ($O_a, O_d, O_s$):** RGB vectors. Simplified implementations may assume $O_a = O_d$ and $O_s$ is a blend of $O_d$ and white [S1].

**Edge Cases and Pitfalls:**
*   **Back-facing surfaces:** The diffuse term must be clamped to 0 when $N \cdot L < 0$ to prevent light from shining through the back of the object [S7].
*   **Non-physical results:** The model does not conserve energy; the sum of reflected light can exceed incident light if coefficients are not carefully tuned [S2].
*   **Vector normalization:** Failure to normalize $N, L, V$ before dot products results in incorrect shading intensity [S2].
*   **Coordinate spaces:** Vectors must be in the same space. Common errors include mixing model-space normals with world-space light directions [S2].

## Subtopics

### Blinn-Phong Shading Model
Uses the halfway vector $H = \frac{L+V}{|L+V|}$ instead of the reflection vector $R$. It is computationally cheaper (no reflection vector calculation) and produces slightly different highlight shapes. Choose when performance is critical or the specific highlight shape of Blinn-Phong is desired [2239-blinn-phong-shading-model.md].

### Blinn-Phong Lighting with Depth Cueing
Extends Blinn-Phong with depth-based fading to simulate atmospheric effects. Choose for scenes requiring depth-based visibility cues [2448-blinn-phong-lighting-with-depth-cueing.md].

### Phong Shading Model for Volume Data
Adapts the Phong model for volume rendering by using gradient vectors as normals and lookup tables for illumination. Choose for ray-casted volume visualization [1353-phong-shading-model-for-volume-data.md].

### Specular Reflections via Polynomial Expansion
Approximates the specular term using polynomial expansions for faster computation. Choose when hardware does not support efficient power functions or for specific optimization constraints [1172-specular-reflections-via-polynomial-expansion.md].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per light source per fragment. The cost is dominated by a few dot products, a power function, and vector arithmetic.
*   **Memory Complexity:** $O(1)$ for material and light parameters. No large tables are required for the standard model [S1, S2].
*   **Performance:** Extremely fast, suitable for real-time rendering on CPUs and GPUs [S2].
*   **Strengths:** Simple to implement, computationally inexpensive, visually pleasing for many materials [S2].
*   **Limitations:** Not physically based (does not conserve energy, incorrect falloff for point lights unless manually added), specular highlights are view-dependent in a way that can look unrealistic for rough surfaces, no shadowing or ambient occlusion included [S2, S7].
*   **When to use:** Real-time applications where performance is critical and physical accuracy is not required. Good for stylized rendering or as a baseline lighting model.
*   **When not to use:** Physically based rendering (PBR) pipelines, scenes requiring accurate energy conservation, or when realistic shadowing and global illumination are needed.

## Background
The Phong shading model was developed by Bui Tuong Phong in 1975 [S2]. It was introduced as an empirical model to approximate the way light reflects off surfaces, specifically focusing on the specular highlight. It became a standard in computer graphics due to its simplicity and visual effectiveness. It is widely used in real-time rendering engines, educational contexts, and as a component in more complex lighting pipelines [S1, S2]. The model is often implemented in fragment shaders in OpenGL and Direct3D pipelines [S2].

## Sources
[S1] Computing and Displaying Isosurfaces inR, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
[S2] Real-Time Global Illumination Using OpenGL And Voxel Cone Tracing, https://doi.org/10.48550/arxiv.2104.00618, fetched 2026-10-26
[S3] sungiant/sdf, https://github.com/sungiant/sdf, fetched 2026-10-26 (unofficial)
[S4] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S5] skmhrk1209/Sphere-Tracing-PyTorch3D, https://github.com/skmhrk1209/Sphere-Tracing-PyTorch3D, fetched 2026-10-26 (unofficial)
[S6] The VolumePro Real-Time Ray-Casting System, https://www.merl.com/publications/docs/TR99-19.pdf, fetched 2026-10-26
[S7] GPU Ray Marching for Ambient Occlusion, https://www.scribd.com/document/520755804/imm6392, fetched 2026-10-28 (unofficial)
