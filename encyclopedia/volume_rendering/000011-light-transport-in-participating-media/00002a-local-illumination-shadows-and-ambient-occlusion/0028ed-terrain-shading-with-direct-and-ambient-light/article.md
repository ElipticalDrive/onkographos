# Terrain Shading with Direct and Ambient Light

Also called: Hemispheric Ambient Occlusion Shading

## Overview
This method computes the final color of a terrain surface point by combining surface albedo, direct sunlight (modulated by shadow and cloud shadow), and hemispheric ambient light (sky and ground bounce modulated by ambient occlusion). It is designed for heightfield-based terrain rendering where normals are baked rather than derived from finite differences to avoid facet artifacts. It differs from general volumetric lighting by operating on a 2.5D surface geometry, using a hemispheric approximation for ambient light instead of full spherical integration, and relying on precomputed or trace-based occlusion factors rather than participating media scattering.

## How it works
**Inputs:**
*   Surface position $\mathbf{p} = (p_x, p_y, p_z)$, where $p_y$ is typically height.
*   Surface normal $\mathbf{n} = (n_x, n_y, n_z)$, unit length.
*   Surface albedo $\mathbf{c}_{albedo}$ (RGB).
*   Sun direction $\mathbf{s}$ (unit vector).
*   Sun radiance $\mathbf{c}_{sun}$ (RGB).
*   Sky ambient color $\mathbf{c}_{sky}$ (RGB).
*   Ground ambient color $\mathbf{c}_{ground}$ (RGB).
*   Shadow factor $S \in [0, 1]$ (from shadow ray casting).
*   Cloud shadow factor $C \in [0, 1]$.
*   Ambient occlusion factor $AO \in [0, 1]$.
*   Cell size `field.cell` (for bias calculation).

**Outputs:**
*   Final pixel color $\mathbf{c}_{out}$ (RGB).

**Data Structures:**
*   No complex volumetric data structures are required. The method operates on scalar and vector attributes associated with the terrain mesh vertices or fragments.

**Steps:**

1.  **Calculate Direct Lighting Scale:**
    Compute the dot product between the normal and sun direction:
    $$ NdotL = n_x s_x + n_y s_y + n_z s_z $$
    Calculate a bias value to offset the shadow ray origin to prevent self-shadowing artifacts on coarse heightfields:
    $$ \text{bias} = \text{field.cell} \times 0.5 + 0.5 $$
    The shadow ray origin is $\mathbf{p} + \mathbf{n} \times \text{bias}$ with jitter. The shadow factor $S$ is determined by tracing this ray. The cloud shadow factor $C$ is determined by querying cloud density at $(p_x, p_z, p_y)$.
    Compute the direct light scale:
    $$ \text{lightScale} = \max(0, NdotL) \times S \times C $$

2.  **Calculate Ambient Occlusion:**
    Compute the ambient occlusion factor $AO$ by tracing ambient occlusion from $\mathbf{p}$ in the hemisphere defined by $\mathbf{n}$. This value represents the fraction of the hemisphere occluded by geometry.

3.  **Calculate Hemispheric Ambient Factor:**
    Determine the sky visibility factor based on the normal's Y component (assuming Y is up):
    $$ \text{sky} = 0.5 + 0.5 \times n_y $$
    Consequently, the ground visibility factor is:
    $$ \text{ground} = 1 - \text{sky} = 0.5 - 0.5 \times n_y $$

4.  **Compute Ambient Color:**
    Interpolate between sky and ground ambient colors based on the sky factor, then modulate by occlusion:
    $$ \mathbf{c}_{ambient} = (\mathbf{c}_{sky} \times \text{sky} + \mathbf{c}_{ground} \times \text{ground}) \times AO $$

5.  **Compute Direct Color:**
    Scale the sun radiance by the direct light scale:
    $$ \mathbf{c}_{direct} = \mathbf{c}_{sun} \times \text{lightScale} $$

6.  **Output Final Color:**
    Multiply the albedo by the sum of direct and ambient components:
    $$ \mathbf{c}_{out} = \mathbf{c}_{albedo} \times (\mathbf{c}_{direct} + \mathbf{c}_{ambient}) $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def terrain_shading(p, n, albedo, sun_dir, sun_radiance, sky_ambient, ground_ambient, shadow_factor, cloud_shadow_factor, ao_factor, cell_size):
    # 1. Direct Lighting
    ndotl = dot(n, sun_dir)
    
    # Bias to prevent self-shadowing on coarse heightfields
    bias = cell_size * 0.5 + 0.5
    
    # Shadow ray origin: p + n * bias + jitter
    # shadow_factor is result of trace_shadow(p + n * bias + jitter, sun_dir)
    # cloud_shadow_factor is result of cloud_shadow_at(p.x, p.z, p.y)
    
    light_scale = max(0.0, ndotl) * shadow_factor * cloud_shadow_factor
    
    # 2. Ambient Occlusion
    # ao_factor is result of trace_ambient_occlusion(p, n)
    
    # 3. Hemispheric Ambient Factor
    sky_factor = 0.5 + 0.5 * n.y
    ground_factor = 0.5 - 0.5 * n.y
    
    # 4. Ambient Color
    ambient_color = (sky_ambient * sky_factor + ground_ambient * ground_factor) * ao_factor
    
    # 5. Direct Color
    direct_color = sun_radiance * light_scale
    
    # 6. Final Output
    out_color = albedo * (direct_color + ambient_color)
    
    return out_color
```

**Parameters:**
*   `field.cell`: The grid cell size of the heightfield. Typical values depend on the world scale, e.g., 1.0 to 10.0 units.
*   `bias`: Calculated as `field.cell * 0.5 + 0.5`. This specific value is critical to prevent self-shadowing artifacts on coarse heightfields [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Self-Shadowing:** If the shadow ray origin is not biased sufficiently, self-shadowing artifacts will appear on the terrain surface, especially on coarse heightfields. The bias must be at least half a cell plus a small constant [S1, unofficial].
*   **Normal Interpolation:** Normals should be baked and interpolated smoothly across cells. Using finite differences on a bilinear heightfield results in piecewise constant normals, causing visible facet artifacts at cell edges [S1, unofficial].
*   **Material Masks:** Slope and moisture fields used for material masking must be smoothed or expanded to avoid thin contour stripes and single-cell artifacts [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per pixel for the shading calculation itself, assuming shadow, cloud shadow, and AO factors are precomputed or available. The cost depends on the complexity of the shadow and AO tracing methods used to generate the input factors.
*   **Memory Complexity:** $O(1)$ additional memory per pixel for the shading calculation.
*   **Strengths:** Simple to implement, computationally inexpensive for the shading step, provides visually plausible results for terrain with baked normals.
*   **Limitations:** Does not account for indirect lighting beyond the hemispheric approximation. Shadow and AO quality depend on the underlying tracing methods. Not suitable for complex volumetric scenes or dynamic global illumination.
*   **When to use:** Real-time terrain rendering where performance is critical and high-fidelity global illumination is not required. Suitable for heightfield-based worlds.
*   **When not to use:** Scenes requiring accurate indirect lighting, complex participating media, or dynamic global illumination.

## Background
This method is derived from the terrain shading implementation in the `theme1` project by mingderwang [S1, unofficial]. It addresses the specific challenges of rendering heightfield-based terrain, such as the need for smooth normals and effective biasing for shadow rays. The hemispheric ambient occlusion approach is a common simplification in real-time graphics for approximating ambient light on surfaces.

## Sources
[S1] theme1, https://github.com/mingderwang/theme1, fetched 2026-10-27 (unofficial)
