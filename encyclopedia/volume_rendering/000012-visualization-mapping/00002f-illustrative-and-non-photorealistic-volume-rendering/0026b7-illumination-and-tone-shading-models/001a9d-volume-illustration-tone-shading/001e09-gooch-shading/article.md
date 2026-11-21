# Gooch Shading
Also called: Central Difference Normal Shading, Gooch Shading with Central Difference Normals, Gooch Tone Shading, Phong/Tone Shading

## Overview
Gooch Shading is a non-photorealistic illumination model for volume rendering that maps surface orientation relative to a light source to a warm-to-cool color gradient to convey shape [S1][S4, unofficial]. It is distinguished from standard Gooch shading by its specific integration with on-the-fly central difference normal estimation in rasterization pipelines, ensuring consistent visual output across different extraction methods (rasterization vs. ray-marching) [S2][S3, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Scalar volume data $V(x,y,z)$, light direction vector $\mathbf{L}$, warm color $C_w$, cool color $C_c$, warm/cool ratios $k_w, k_c$, base ray sample color $I_r$, and opacity $\alpha$ [S4, unofficial].
- **Output:** Final fragment color $c$ [S4, unofficial].

**Data Structures:**
- **Vertex Normals:** Computed on the fly using central differences on the scalar field [S2][S3, unofficial]. No pre-computed normal buffer is required for the rasterization approach [S2].

**Steps:**
1. **Normal Estimation:** For each vertex or sample point on the isosurface, compute the gradient $\mathbf{G}$ using central differences: $\mathbf{G} = [\frac{\partial V}{\partial x}, \frac{\partial V}{\partial y}, \frac{\partial V}{\partial z}]$, where $\frac{\partial V}{\partial x} \approx \frac{V(x+1, y, z) - V(x-1, y, z)}{2}$, and similarly for $y$ and $z$ [S2][S3, unofficial]. Normalize $\mathbf{G}$ to unit length [S4, unofficial].
2. **Pass to Fragment Shader:** The computed normal $\mathbf{G}$ and normalized light direction $\mathbf{L}$ are passed to the fragment shader [S2][S3, unofficial].
3. **Dot Product Factor:** Compute $\kappa = \mathbf{G} \cdot \mathbf{L}$ [S4, unofficial]. The illuminated contribution uses only the positive dot product, becoming zero at orthogonal orientations [S4, unofficial].
4. **Tone Contribution:** Calculate $I_t$ by interpolating between $C_w$ and $C_c$ based on $\kappa$: $I_t = \left(k_w \frac{1.0 + \kappa}{2}\right) C_w + \left(k_c \left(1 - \frac{1.0 + \kappa}{2}\right)\right) C_c$ [S4, unofficial].
5. **Final Color:** Combine the tone contribution with the base ray sample color $I_r$ and opacity $\alpha$: $c = I_t \alpha + I_r (1 - \alpha)$ [S4, unofficial]. For multiple light sources, sum over all $NL$ lights: $c = \sum_{i=1}^{NL} (I_{ti} \alpha + I_{ri} (1 - \alpha))$ [S4, unofficial].

**Parameters:**
- **Warm Color ($C_w$):** Typically yellow, defined as RGB $(R_w, G_w, B_w)$ [S4, unofficial].
- **Cool Color ($C_c$):** Typically blue, defined as RGB $(R_c, G_c, B_c)$ [S4, unofficial].
- **Ratios ($k_w, k_c$):** Weights for warm and cool color contribution to the final result [S4, unofficial].

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** Central difference requires neighboring voxels; boundary handling is not specified in the sources [G].
- **Degenerate Normals:** If the scalar field is flat (zero gradient), the normal is undefined [G].
- **Orthogonal Orientations:** The dot product $\kappa$ naturally handles values between -1 and 1, but the illuminated contribution becomes zero at orthogonal orientations ($\kappa = 0$) [S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_central_difference_normal(volume, x, y, z):
    # Central difference gradient approximation
    dx = (volume[x + 1, y, z] - volume[x - 1, y, z]) / 2.0
    dy = (volume[x, y + 1, z] - volume[x, y - 1, z]) / 2.0
    dz = (volume[x, y, z + 1] - volume[x, y, z - 1]) / 2.0
    length = (dx*dx + dy*dy + dz*dz) ** 0.5
    if length == 0.0:
        return vec3(0.0, 0.0, 0.0) # Degenerate case: flat field
    return vec3(dx / length, dy / length, dz / length)

def gooche_shading_fragment(G, L, C_w, C_c, k_w, k_c, alpha, I_r):
    # Compute dot product factor kappa, clamped to positive values per source
    kappa = max(0.0, dot(G, L))
    # Calculate tone contribution interpolation factor
    t = (1.0 + kappa) / 2.0
    # Interpolate between warm and cool colors
    I_t = (k_w * t) * C_w + (k_c * (1.0 - t)) * C_c
    # Blend tone contribution with base ray sample color using opacity
    c = I_t * alpha + I_r * (1.0 - alpha)
    return c
```

## Operating characteristics
- **Time Complexity:** Normal computation is $O(1)$ per vertex/sample [G]. Shading is $O(1)$ per fragment [G].
- **Memory Complexity:** No additional memory for normals if computed on the fly [S2].
- **Measured Performance:** Not given in the sources.
- **Strengths:** Consistent visual appearance across different rendering techniques (rasterization and ray-marching) due to identical fragment shader implementation [S2][S3, unofficial].
- **Limitations:** Relies on central difference normal estimation, which requires neighboring voxels and may produce undefined normals in flat regions [G]. Boundary handling is not specified in the sources [G].

## Background
Gooch Shading was originally proposed by Gooch et al. in 1998 [S1][S2]. It is used in volume rendering to provide illustrative shading that enhances shape perception by mapping orientation to warm and cool tones [S1][S4, unofficial]. The specific application with central difference normals in rasterization is described in [S2][S3, unofficial].

## Sources
[S1] Volume illustration for medicine and flows, https://doi.org/10.1145/1185657.1185694, fetched 2026-10-28
[S2] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
[S3] Efficient Direct Isosurface Rasterization of Scalar Volumes - EG, https://diglib.eg.org/items/8a65e0f0-c048-4fce-9d0b-5acf07452485, fetched 2026-10-30 (unofficial)
[S4] Implementation of Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2005/Thurner-2005-IVR/Thurner-2005-IVR-paper.pdf, fetched 2026-10-29 (unofficial)
