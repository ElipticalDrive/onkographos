# Physically-Based Rendering (PBR) Lighting
Also called: Burley BRDF, Disney PBR, Disney PBR Lighting, Disney-GGX BRDF Model, Metallic-Roughness PBR, PBR Material Shading, Physically-Based Rendering

## Overview
Calculates direct lighting contributions for surface points using a physically-based shading model that separates diffuse and specular components, driven by metallic-roughness material inputs [S2, unofficial]. It differs from its parent [Direct Lighting and Shading](3953-direct-lighting-and-shading.md) by replacing the non-standard GGX-like specular term and Lambertian diffuse with the Burley diffuse model and a microfacet specular model, providing energy-conserving, view-dependent shading [S1, S3].

## How it works
**Inputs and Outputs**
- **Inputs**: `baseColor` (vec3, RGB), `mrSample` (vec4, RGBA), surface normal `n` (vec3), view direction `v` (vec3), light direction `l` (vec3) [S2, unofficial].
- **Output**: `PBRInfo` struct containing precomputed dot products and material parameters for the shading evaluation [S2, unofficial].

**Data Structures**
The `PBRInfo` struct holds the following fields:
- `NdotL`: float
- `NdotV`: float
- `NdotH`: float
- `LdotH`: float
- `VdotH`: float
- `perceptualRoughness`: float
- `reflectance0`: vec3
- `reflectance90`: vec3
- `alphaRoughness`: float
- `diffuseColor`: vec3
- `specularColor`: vec3
- `n`: vec3
- `v`: vec3 [S2, unofficial]

**Steps**
1. **Extract Material Properties**:
   - Extract `perceptualRoughness` from `mrSample.g` (green channel) [S2, unofficial].
   - Extract `metallic` from `mrSample.b` (blue channel) [S2, unofficial].
   - Clamp `perceptualRoughness` to `[c_MinRoughness, 1.0]`, where `c_MinRoughness = 0.04` [S2, unofficial].
   - Clamp `metallic` to `[0.0, 1.0]` [S2, unofficial].

2. **Calculate Alpha Roughness**:
   - Compute `alphaRoughness = perceptualRoughness * perceptualRoughness` [S2, unofficial].

3. **Calculate Diffuse Color**:
   - Define `f0 = vec3(0.04)` [S2, unofficial].
   - Compute `diffuseColor = baseColor.rgb * (vec3(1.0) - f0)` [S2, unofficial].
   - Apply metallic factor: `diffuseColor *= 1.0 - metallic` [S2, unofficial].

4. **Calculate Specular Color**:
   - Compute `specularColor = mix(f0, baseColor.rgb, metallic)` [S2, unofficial].

5. **Calculate Grazing Reflectance**:
   - Determine `reflectance` as the maximum component of `specularColor` [S2, unofficial].
   - Compute `reflectance90 = clamp(reflectance * 25.0, 0.0, 1.0)` [S2, unofficial].
   - Set `reflectance0 = specularColor` [S2, unofficial].

6. **Compute Dot Products**:
   - Compute `NdotL = dot(n, l)` [G].
   - Compute `NdotV = clamp(abs(dot(n, v)), 0.001, 1.0)` [S2, unofficial].
   - Compute half-vector `h = normalize(l + v)` [G].
   - Compute `NdotH = dot(n, h)` [G].
   - Compute `LdotH = dot(l, h)` [G].
   - Compute `VdotH = dot(v, h)` [G].

7. **Populate PBRInfo**:
   - Fill the `PBRInfo` struct with the calculated values: `NdotL`, `NdotV`, `NdotH`, `LdotH`, `VdotH`, `perceptualRoughness`, `reflectance0`, `reflectance90`, `alphaRoughness`, `diffuseColor`, `specularColor`, `n`, `v` [S2, unofficial].

**Shading Evaluation**
Once `PBRInfo` is populated, the shading contributions are calculated as follows:

1. **Diffuse Contribution (Burley Model)**:
   - Compute `f90 = 2.0 * LdotH * LdotH * alphaRoughness - 0.5` [S2, unofficial].
   - Compute diffuse term:
     $$
     \text{diffuse} = \frac{\text{diffuseColor}}{\pi} \times (1.0 + f90 \times (1.0 - \text{NdotL})^5) \times (1.0 + f90 \times (1.0 - \text{NdotV})^5)
     $$
     [S2, unofficial]

2. **Specular Fresnel Term**:
   - Compute:
     $$
     F = \text{reflectance0} + (\text{reflectance90} - \text{reflectance0}) \times \text{clamp}(1.0 - \text{VdotH}, 0.0, 1.0)^5
     $$
     [S2, unofficial]

3. **Geometric Attenuation (Schlick-GGX)**:
   - Compute `rSqr = alphaRoughness * alphaRoughness` [S2, unofficial].
   - Compute attenuation for light:
     $$
     \text{attenuationL} = \frac{2.0 \times \text{NdotL}}{\text{NdotL} + \sqrt{\text{rSqr} + (1.0 - \text{rSqr}) \times \text{NdotL}^2}}
     $$
     [S2, unofficial]
   - Compute attenuation for view:
     $$
     \text{attenuationV} = \frac{2.0 \times \text{NdotV}}{\text{NdotV} + \sqrt{\text{rSqr} + (1.0 - \text{rSqr}) \times \text{NdotV}^2}}
     $$
     [S2, unofficial]
   - Return `attenuationL * attenuationV` [S2, unofficial].

4. **Microfacet Distribution (Trowbridge-Reitz)**:
   - Compute `roughnessSq = alphaRoughness * alphaRoughness` [S2, unofficial].
   - Compute denominator factor:
     $$
     f = (\text{NdotH} \times \text{roughnessSq} - \text{NdotH}) \times \text{NdotH} + 1.0
     $$
     [S2, unofficial]
   - Return distribution:
     $$
     D = \frac{\text{roughnessSq}}{\pi \times f^2}
     $$
     [S2, unofficial]

**Parameters**
- `M_PI`: 3.141592653589793 [S2, unofficial].
- `c_MinRoughness`: 0.04 [S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def vec3(r, g=0.0, b=0.0):
    return [r, g, b]

def dot(a, b):
    return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]

def normalize(v):
    mag = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
    if mag == 0.0: return [0.0, 0.0, 0.0]
    return [v[0]/mag, v[1]/mag, v[2]/mag]

def mix(a, b, t):
    return [a[i] + t*(b[i] - a[i]) for i in range(3)]

def clamp(x, lo, hi):
    return max(lo, min(hi, x))

def calculate_pbr_info(base_color, mr_sample, n, v, l):
    # 1. Extract and clamp material properties
    perceptual_roughness = mr_sample[1]
    metallic = mr_sample[3]
    
    c_min_roughness = 0.04
    perceptual_roughness = clamp(perceptual_roughness, c_min_roughness, 1.0)
    metallic = clamp(metallic, 0.0, 1.0)
    
    # 2. Calculate alpha roughness
    alpha_roughness = perceptual_roughness * perceptual_roughness
    
    # 3. Calculate diffuse color
    f0 = vec3(0.04)
    diffuse_color = [base_color[i] * (1.0 - f0[i]) for i in range(3)]
    diffuse_color = [diffuse_color[i] * (1.0 - metallic) for i in range(3)]
    
    # 4. Calculate specular color
    specular_color = mix(f0, base_color, metallic)
    
    # 5. Calculate grazing reflectance
    reflectance = max(specular_color)
    reflectance90 = clamp(reflectance * 25.0, 0.0, 1.0)
    reflectance0 = specular_color
    
    # 6. Compute dot products
    n_dot_l = dot(n, l)
    n_dot_v = clamp(abs(dot(n, v)), 0.001, 1.0)
    
    h = normalize([l[i] + v[i] for i in range(3)])
    n_dot_h = dot(n, h)
    l_dot_h = dot(l, h)
    v_dot_h = dot(v, h)
    
    # 7. Populate PBRInfo struct
    pbr_info = {
        'NdotL': n_dot_l,
        'NdotV': n_dot_v,
        'NdotH': n_dot_h,
        'LdotH': l_dot_h,
        'VdotH': v_dot_h,
        'perceptualRoughness': perceptual_roughness,
        'reflectance0': reflectance0,
        'reflectance90': reflectance90,
        'alphaRoughness': alpha_roughness,
        'diffuseColor': diffuse_color,
        'specularColor': specular_color,
        'n': n,
        'v': v
    }
    
    return pbr_info

def evaluate_diffuse(pbr_info):
    l_dot_h = pbr_info['LdotH']
    alpha_roughness = pbr_info['alphaRoughness']
    n_dot_l = pbr_info['NdotL']
    n_dot_v = pbr_info['NdotV']
    diffuse_color = pbr_info['diffuseColor']
    
    f90 = 2.0 * l_dot_h * l_dot_h * alpha_roughness - 0.5
    
    term1 = 1.0 + f90 * math.pow(1.0 - n_dot_l, 5.0)
    term2 = 1.0 + f90 * math.pow(1.0 - n_dot_v, 5.0)
    
    return [(diffuse_color[i] / math.pi) * term1 * term2 for i in range(3)]

def evaluate_fresnel(pbr_info):
    reflectance0 = pbr_info['reflectance0']
    reflectance90 = pbr_info['reflectance90']
    v_dot_h = pbr_info['VdotH']
    
    f = [reflectance0[i] + (reflectance90[i] - reflectance0[i]) * math.pow(clamp(1.0 - v_dot_h, 0.0, 1.0), 5.0) for i in range(3)]
    return f

def evaluate_geometry(pbr_info):
    alpha_roughness = pbr_info['alphaRoughness']
    n_dot_l = pbr_info['NdotL']
    n_dot_v = pbr_info['NdotV']
    
    r_sqr = alpha_roughness * alpha_roughness
    
    denom_l = n_dot_l + math.sqrt(r_sqr + (1.0 - r_sqr) * n_dot_l * n_dot_l)
    attenuation_l = (2.0 * n_dot_l) / denom_l if denom_l != 0 else 0.0
    
    denom_v = n_dot_v + math.sqrt(r_sqr + (1.0 - r_sqr) * n_dot_v * n_dot_v)
    attenuation_v = (2.0 * n_dot_v) / denom_v if denom_v != 0 else 0.0
    
    return attenuation_l * attenuation_v

def evaluate_distribution(pbr_info):
    alpha_roughness = pbr_info['alphaRoughness']
    n_dot_h = pbr_info['NdotH']
    
    roughness_sq = alpha_roughness * alpha_roughness
    f = (n_dot_h * roughness_sq - n_dot_h) * n_dot_h + 1.0
    
    return roughness_sq / (math.pi * f * f)
```

**Edge Cases and Pitfalls**
- **Precision**: `NdotV` is clamped to a minimum of 0.001 to avoid division by zero or instability in the geometric term when the view vector is nearly perpendicular to the normal [S2, unofficial].
- **Boundaries**: Roughness is clamped to a minimum of 0.04 (`c_MinRoughness`) to prevent numerical issues associated with perfectly smooth surfaces in this specific approximation [S2, unofficial].
- **Degenerate Input**: If `l + v` is zero (light and view directions are opposite), the half-vector `h` is undefined [G]. The pseudocode assumes `l + v` is non-zero; in practice, this case implies the light is behind the viewer, which may result in zero lighting contribution anyway [G].

## Operating characteristics
Not given in the sources.

## Background
This method is derived from the Disney PBR shading model, specifically the implementation found in the "cgue22-greed" project repository [S2, unofficial]. It is based on microfacet theory, which represents real-world materials using BRDF, normal distribution, geometry, and Fresnel terms [S1]. The model splits the BRDF into a diffuse term using the Burley approximation and a specular term using the Trowbridge-Reitz normal distribution function [S3]. It is used in the context of direct volume rendering to calculate lighting at surface intersections determined by ray marching and iso-surface extraction [S2, unofficial].

## Sources
[S1] Jrender: An efficient differentiable rendering library based on Jittor, https://doi.org/10.1016/j.gmod.2023.101202, fetched 2026-10-27
[S2] Masdaofdisasda/Vis1_Direct_Volume_Rendering_2022W, https://github.com/Masdaofdisasda/Vis1_Direct_Volume_Rendering_2022W, fetched 2026-10-26 (unofficial)
[S3] Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering, https://doi.org/10.1007/978-3-031-73390-1_2, fetched 2026-10-27
