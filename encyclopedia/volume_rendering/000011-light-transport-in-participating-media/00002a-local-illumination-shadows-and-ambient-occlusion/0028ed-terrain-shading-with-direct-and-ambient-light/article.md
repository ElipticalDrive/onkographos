# Terrain Shading with Direct and Ambient Light

Also called: Hemispheric Ambient Occlusion Shading

## Overview
This method computes the final color of a terrain surface point by combining albedo, direct sunlight (modulated by shadow and cloud shadow), and hemispheric ambient light (sky and ground bounce modulated by ambient occlusion). It is designed for heightfield-based terrain rendering where normals are baked rather than derived from finite differences to avoid facet artifacts. It differs from general volume shading by treating the terrain as a boundary surface with local illumination, relying on precomputed or efficiently sampled ambient occlusion factors rather than full volumetric light transport.

## How it works
**Inputs:**
- Surface position $\mathbf{p} = (px, py, pz)$.
- Surface normal $\mathbf{n} = (nx, ny, nz)$, assumed unit length.
- Albedo color $\mathbf{c}_{albedo}$.
- Sun direction $\mathbf{l}_{sun}$ (normalized).
- Sun radiance $\mathbf{c}_{sun}$.
- Sky ambient color $\mathbf{c}_{sky}$.
- Ground ambient color $\mathbf{c}_{ground}$.
- Shadow factor $S \in [0, 1]$ (from shadow casting).
- Cloud shadow factor $C \in [0, 1]$.
- Ambient occlusion factor $AO \in [0, 1]$.

**Outputs:**
- Final color $\mathbf{c}_{out}$.

**Data Structures:**
- Vectors are 3-component floats.
- Colors are 3-component floats (RGB).

**Steps:**

1.  **Normal Dot Light:** Calculate the dot product between the normal and the sun direction.
    $$ N \cdot L = n_x l_x + n_y l_y + n_z l_z $$

2.  **Shadow Bias:** To prevent self-shadowing artifacts on coarse heightfields, calculate a bias offset. The bias is defined as:
    $$ \text{bias} = \text{field.cell} \times 0.5 + 0.5 $$
    [S1, unofficial]

3.  **Shadow Tracing:** If $N \cdot L > 0$, trace a shadow ray from a biased starting position to determine if the point is in shadow. The start position is:
    $$ \mathbf{p}_{start} = \mathbf{p} + \mathbf{n} \times \text{bias} $$
    Jitter is applied to the trace to reduce aliasing. [S1, unofficial]
    *Note: The shadow factor $S$ is the result of this trace (1 if visible, 0 if occluded, or intermediate if soft shadows).*

4.  **Cloud Shadow:** Retrieve the cloud shadow factor $C$ at the surface position $(px, pz)$ and height $py$. [S1, unofficial]

5.  **Direct Light Scale:** Calculate the intensity scale for direct sunlight.
    $$ \text{lightScale} = \max(0, N \cdot L) \times S \times C $$

6.  **Ambient Occlusion:** Calculate the ambient occlusion factor $AO$ at $\mathbf{p}$. This typically involves tracing rays into the hemisphere or using a precomputed field. [S1, unofficial]

7.  **Hemispheric Ambient Factor:** Calculate the sky visibility factor based on the normal's Y component.
    $$ \text{sky} = 0.5 + 0.5 \times n_y $$
    This linearly interpolates between full ground visibility ($n_y = -1 \implies \text{sky}=0$) and full sky visibility ($n_y = 1 \implies \text{sky}=1$). [S1, unofficial]

8.  **Ambient Color:** Compute the ambient color by interpolating between sky and ground ambient colors, then modulating by AO.
    $$ \mathbf{c}_{ambient} = \left( \mathbf{c}_{sky} \times \text{sky} + \mathbf{c}_{ground} \times (1 - \text{sky}) \right) \times AO $$
    [S1, unofficial]

9.  **Direct Color:** Compute the direct sunlight contribution.
    $$ \mathbf{c}_{direct} = \mathbf{c}_{sun} \times \text{lightScale} $$

10. **Final Color:** Combine direct and ambient contributions, modulated by albedo.
    $$ \mathbf{c}_{out} = \mathbf{c}_{albedo} \times (\mathbf{c}_{direct} + \mathbf{c}_{ambient}) $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def shade_terrain(p, n, albedo, sun_dir, sun_radiance, sky_ambient, ground_ambient, shadow_factor, cloud_shadow_factor, ao_factor):
    # 1. Normal Dot Light
    ndotl = n.x * sun_dir.x + n.y * sun_dir.y + n.z * sun_dir.z
    
    # 2. Shadow Bias
    # field.cell is the grid cell size; 0.5 is an arbitrary constant for bias
    bias = field.cell * 0.5 + 0.5
    
    # 3. Shadow Tracing (if sun is above horizon)
    shadow = 1.0
    if ndotl > 0:
        # Start position biased along normal
        start_pos = p + n * bias
        # Trace shadow ray with jitter
        # shadow_factor is the result of this trace
        shadow = shadow_factor 
        
    # 4. Cloud Shadow
    # cloud_shadow_at takes x, z, y coordinates
    cloud_shade = cloud_shadow_at(p.x, p.z, p.y)
    
    # 5. Direct Light Scale
    light_scale = max(0.0, ndotl) * shadow * cloud_shade
    
    # 6. Ambient Occlusion
    # ao_factor is the result of traceAmbientOcclusion
    ao = ao_factor
    
    # 7. Hemispheric Ambient Factor
    sky_factor = 0.5 + 0.5 * n.y
    
    # 8. Ambient Color
    # Interpolate between sky and ground ambient
    sky_contrib = sky_ambient * sky_factor
    ground_contrib = ground_ambient * (1.0 - sky_factor)
    ambient_color = (sky_contrib + ground_contrib) * ao
    
    # 9. Direct Color
    direct_color = sun_radiance * light_scale
    
    # 10. Final Color
    out_color = albedo * (direct_color + ambient_color)
    
    return out_color
```

**Parameters:**
- `field.cell`: Grid cell size. Typical values depend on the terrain resolution and world scale.
- `bias` constant: `0.5` is used in the source implementation. [S1, unofficial]

**Edge Cases and Pitfalls:**
- **Self-Shadowing:** If the shadow ray origin is not biased sufficiently, self-shadowing artifacts (acne) will appear on coarse heightfields. The bias must be at least half a cell plus a small constant. [S1, unofficial]
- **Normal Accuracy:** Using finite differences on a bilinear heightfield results in piecewise constant normals, causing facet artifacts. Normals should be baked or interpolated smoothly. [S1, unofficial]
- **Boundary Reads:** Sampling at tile boundaries can cause out-of-bounds reads if not handled. Ensure sampling logic clamps or wraps coordinates to avoid NaN values. [S1, unofficial]

## Operating characteristics
- **Time Complexity:** $O(1)$ per pixel for the shading calculation itself, assuming shadow, cloud, and AO factors are precomputed or efficiently sampled. If shadow rays are traced per pixel, complexity depends on the number of shadow samples.
- **Memory Complexity:** $O(1)$ additional memory per pixel for shading. Requires storage for terrain heightfield, normal map, shadow maps, and cloud fields.
- **Strengths:** Fast, visually plausible results for terrain. Handles sky/ground bounce simply. Avoids facet artifacts with baked normals.
- **Limitations:** Approximate shadows and AO. Does not handle complex volumetric lighting or global illumination beyond hemispheric ambient. Single-tile limitation causes edge artifacts. [S1, unofficial]
- **When to use:** Real-time terrain rendering where performance is critical and high-fidelity global illumination is not required.
- **When not to use:** Scenes requiring accurate soft shadows, complex indirect lighting, or multi-tile seamless terrain.

## Background
This method is derived from the shading pipeline of the `theme1` project by mingderwang, which implements a procedural terrain renderer. It emphasizes baked normals and smooth material masks to avoid artifacts common in simple heightfield shading. The approach combines direct sunlight with a simplified hemispheric ambient model modulated by ambient occlusion. [S1, unofficial]

## Sources
[S1] theme1, https://github.com/mingderwang/theme1, fetched 2026-10-27 (unofficial)
