# Sky Radiance Model

Also called: atmospheric scattering

## Overview
The Sky Radiance Model provides background illumination by sampling a precomputed sky view texture and adding a discrete sun disk contribution based on the ray direction relative to the sun direction [S1, unofficial]. It solves the problem of generating sky lighting without performing expensive real-time atmospheric scattering integrals, relying instead on a baked lookup texture [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Ray direction $\mathbf{d}$ (normalized), sun direction $\mathbf{s}$ (normalized), sun color $\mathbf{C}_{sun}$, sky view texture $\mathcal{T}_{sky}$, solar angular radius.
*   **Output:** Radiance $\mathbf{L}_{sky}$ (RGB vector).

**Data Structures**
*   **Sky View Texture ($\mathcal{T}_{sky}$):** A 2D texture of size $W \times H$ (default $400 \times 400$) storing RGB radiance values [S1, unofficial].
*   **Sun Direction ($\mathbf{s}$):** A normalized 3D vector representing the sun's position [S1, unofficial].
*   **Sun Color ($\mathbf{C}_{sun}$):** A 4D vector `float4(100.f, 100.f, 100.f, 0.f)` where the RGB components define intensity and the W component is zero [S1, unofficial].

**Parameters**
*   `VX_SKY_LUT_WIDTH`: 400 [S1, unofficial].
*   `VX_SKY_LUT_HEIGHT`: 400 [S1, unofficial].
*   `VX_SKY_SOLAR_RADIUS_RAD`: Angular radius of the sun disk in radians [S1, unofficial].
*   `VX_SKY_PLANET_RADIUS`: 6371e3f [S1, unofficial].
*   `VX_SKY_PLANET_CENTER`: `float3(0.f, -VX_SKY_PLANET_RADIUS, 0.f)` [S1, unofficial].
*   `VX_SKY_HEIGHT`: 100e3f [S1, unofficial].
*   `VX_SKY_RAYLEIGH_HEIGHT`: `VX_SKY_HEIGHT * 0.08f` [S1, unofficial].
*   `VX_SKY_MIE_HEIGHT`: `VX_SKY_HEIGHT * 0.012f` [S1, unofficial].
*   `VX_SKY_RAYLEIGH_S`: `float3(5.802e-6f, 13.558e-6f, 33.100e-6f)` [S1, unofficial].
*   `VX_SKY_MIE_S`: `float3(3.996e-6f, 3.996e-6f, 3.996e-6f)` [S1, unofficial].
*   `VX_SKY_MIE_A`: `float3(4.40e-6f, 4.40e-6f, 4.40e-6f)` [S1, unofficial].
*   `VX_SKY_OZONE_A`: `float3(0.650e-6f, 1.881e-6f, 0.085e-6f)` [S1, unofficial].

**Steps**
1.  Convert ray direction $\mathbf{d}$ to 2D UV coordinates $(u, v)$ using a helper function `sky_view_ray_dir_to_uv` [S1, unofficial].
2.  Sample the sky view texture at $(u, v)$ using a linear sampler at mip level 0 to obtain base radiance $\mathbf{L}_{base}$ [S1, unofficial]:
    $$ \mathbf{L}_{base} = \mathcal{T}_{sky}.SampleLevel(\mathbf{d}_{uv}, 0.0).rgb $$
3.  Compute the cosine of the solar radius:
    $$ \cos \theta_{max} = \cos(\text{VX\_SKY\_SOLAR\_RADIUS\_RAD}) $$
4.  Check sun disk intersection: if $\mathbf{d} \cdot \mathbf{s} \ge \cos \theta_{max}$, add the sun contribution to the radiance [S1, unofficial]:
    $$ \mathbf{L}_{sky} = \mathbf{L}_{base} + \frac{\mathbf{C}_{sun}.rgb}{\sqrt{1.0 - \cos \theta_{max}}} $$
    Otherwise, $\mathbf{L}_{sky} = \mathbf{L}_{base}$ [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_sky_radiance(ray_dir: tuple, sun_dir: tuple, sun_color: tuple, solar_radius_rad: float) -> tuple:
    # sky_view_ray_dir_to_uv is not defined in the sources [S1, unofficial]
    # Assuming standard equirectangular projection for implementation:
    u = 0.5 + math.atan2(ray_dir[2], ray_dir[0]) / (2.0 * math.pi)
    v = 0.5 - math.asin(ray_dir[1]) / math.pi
    
    # Sample sky texture at (u, v) using linear filtering at mip level 0 [S1, unofficial]
    # base_radiance = sky_tex.sample_level((u, v), 0.0).rgb
    base_radiance = (0.0, 0.0, 0.0)  # Not given in the sources.
    
    cos_theta_max = math.cos(solar_radius_rad)
    dot_product = ray_dir[0]*sun_dir[0] + ray_dir[1]*sun_dir[1] + ray_dir[2]*sun_dir[2]
    
    if dot_product >= cos_theta_max:
        inv_sqrt = 1.0 / math.sqrt(1.0 - cos_theta_max)
        sun_contribution = (sun_color[0] * inv_sqrt, sun_color[1] * inv_sqrt, sun_color[2] * inv_sqrt)
        final_radiance = (base_radiance[0] + sun_contribution[0], base_radiance[1] + sun_contribution[1], base_radiance[2] + sun_contribution[2])
    else:
        final_radiance = base_radiance
        
    return final_radiance
```

**Edge Cases and Pitfalls**
*   **UV Mapping:** The function `sky_view_ray_dir_to_uv` is not defined in the sources [S1, unofficial]. Implementers must ensure the UV mapping matches the coordinate system used to generate the sky view texture.
*   **Division by Zero:** If `VX_SKY_SOLAR_RADIUS_RAD` is zero, the term $\sqrt{1.0 - \cos \theta_{max}}$ approaches zero, potentially causing division by zero or numerical instability [S1, unofficial]. The sources do not specify a guard for this case.

## Subtopics

### [Sky Atmosphere](4899-sky-atmosphere.md)
An Earth-like atmosphere model that bakes single-scattered sunlight (Rayleigh, Mie, and ozone scattering) into an equirectangular map sampled by `environment_light`, with sun irradiance attenuated by air along its direction. It differs from the base Sky Radiance Model by explicitly modeling spectral attenuation and scattering physics rather than relying solely on a precomputed view texture and simple sun disk addition.

### Uniform Sky
A constant environment lighting term baked into a uniform sky parameter [S2, unofficial]. It is chosen for simplicity and performance when directional sky variation is not needed, serving as a fallback or base ambient term [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per ray, dominated by texture sampling and arithmetic operations [S1, unofficial].
*   **Memory Complexity:** $O(W \times H)$ for the sky view texture, where $W=400$ and $H=400$ by default [S1, unofficial].
*   **Performance:** Not given in the sources.
*   **Strengths:** Simple implementation, low memory footprint, provides directional sky lighting [S1, unofficial].
*   **Limitations:** Relies on a precomputed texture, limiting dynamic sky changes without regeneration [S1, unofficial].
*   **When to Use:** Not given in the sources.
*   **When Not to Use:** Not given in the sources.

## Background
The model appears in a pull request for a PBR renderer aiming for a MagicaVoxel look, where it is combined with a uniform sky and an infinite ground plane [S2, unofficial]. The renderer adds the sun radiance to the escaped radiance from the sky model [S2, unofficial]. It is part of the VXRay renderer ecosystem [S1, unofficial].

## Sources
[S1] Nelarius/vxray, https://github.com/Nelarius/vxray, fetched 2026-10-24 (unofficial)
[S2] Milestone 13: MagicaVoxel look (.vox, DDA voxel grid, sun/sky, bloom, style voxel), https://github.com/adriendjn/mypbr/pull/10, fetched 2026-10-25 (unofficial)
