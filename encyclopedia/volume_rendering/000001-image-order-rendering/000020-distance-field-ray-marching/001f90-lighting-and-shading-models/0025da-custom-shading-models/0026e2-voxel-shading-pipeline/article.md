# Voxel Shading Pipeline

Also called: Block Shading, Shading and Lighting, Voxel Lighting

## Overview
The Voxel Shading Pipeline is a fragment-shader-based lighting model for ray-marched voxel worlds that combines Lambertian diffuse shading, rim lighting, indirect ambient reflection, and a compact microfacet specular term [S1, unofficial]. It solves the problem of providing visually plausible surface lighting for discrete voxel geometry without requiring explicit mesh generation or complex global illumination [S2, unofficial]. It differs from sibling methods by integrating hard shadow casting and emissive block handling directly into the ray-marched hit loop, whereas [GGX Microfacet BRDF](3715-ggx-microfacet-brdf.md) focuses on analytical surface scattering and [Fast Shading for Voxel Engines](5606-fast-shading-for-voxel-engines.md) optimizes for engine integration [S1, S2, unofficial].

## How it works
**Inputs:** Camera position (`camPos`), ray direction (`rayDir`), hit position (`hitPosition`), hit cell coordinates (`hitCell`), hit face normal (`hitFaceNormal`), block type (`b`), sun direction (`L`), and time (`time`) for sun rotation.
**Outputs:** Final pixel color (`finalColor`).

**Data Structures:**
*   `TraceResult`: Contains `hit` (bool), `t` (float, distance), `cell` (ivec3), `normal` (vec3).
*   `BlockType`: Enum including `BLOCK_EMISSIVE`.

**Parameters:**
*   `lightColor`: `vec3(1.0, 0.96, 0.85)` [S1, unofficial].
*   `ambient`: `vec3(0.12, 0.14, 0.18)` [S1, unofficial].
*   `roughness`: `0.54` [S1, unofficial].
*   `F0`: `0.14` [S1, unofficial].
*   `rimPower`: `2.2` [S1, unofficial].
*   `rimIntensity`: `0.12` [S1, unofficial].
*   `indirectFactor`: `0.25` [S1, unofficial].
*   `emissiveIndirectFactor`: `0.35` [S1, unofficial].
*   `emissiveDirectFactor`: `0.5` [S1, unofficial].
*   `rimSpecularFactor`: `0.6` [S1, unofficial].
*   `toneMapScale`: `2.0` [S1, unofficial].
*   `SUN_ROT_SPEED`: Constant for sun rotation [S1, unofficial].
*   `ENABLE_SHADOW`: Boolean flag [S1, unofficial].
*   `ENABLE_ATMOSPHERE`: Boolean flag [S1, unofficial].

**Steps:**
1.  **Ray Hit Validation:** If no hit, return sky color. If hit distance `t <= 0.0`, set `hitPosition = camPos`; else `hitPosition = camPos + rayDir * (t + 1e-5)` [S1, unofficial].
2.  **Normal and Vector Setup:** Normalize `hitFaceNormal` to get surface normal `N`. Compute view vector `V = normalize(rayDir)`. Compute sun direction `L` based on time [S1, unofficial].
3.  **Atmosphere/Absorption:** Compute `absorpColor` using atmospheric scattering or absorption model [S1, unofficial]. If `ENABLE_ATMOSPHERE`, compute sky color via atmosphere function; else `sky = absorpColor` [S1, unofficial].
4.  **Local Coordinates:** Calculate local position within the voxel: `local = hitPosition - vec3(float(hitCell.x), float(hitCell.y), float(hitCell.z))`. Apply `fract()` to get `localPos` [S1, unofficial].
5.  **Albedo Sampling:** Sample block albedo texture using `hitCell` and `localPos` [S1, unofficial].
6.  **Diffuse Lighting:** Compute `nDotL = max(0.0, dot(N, L))`. Lambertian term `lambert = nDotL` [S1, unofficial].
7.  **Rim Lighting:** Compute `rim = pow(1.0 - max(0.0, dot(N, V)), 2.2) * 0.12` [S1, unofficial].
8.  **Shadow Casting:** Initialize `shadow = 1.0`. If `ENABLE_SHADOW`, trace a secondary ray from `hitPosition + N * 0.01` in direction `L`. If the trace hits a voxel, set `shadow = 0.0` [S1, unofficial].
9.  **Direct and Indirect Light:**
    *   `directLight = lightColor * lambert` [S1, unofficial].
    *   `indirectLight = 0.25 * albedo` [S1, unofficial].
10. **Emissive Handling:** If block type is `BLOCK_EMISSIVE`:
    *   Get emission color and intensity.
    *   `indirectLight += 0.35 * emission` [S1, unofficial].
    *   `directLight += emission * 0.5` [S1, unofficial].
11. **Lighting Integration:** Compute `diffAlbedo = albedo * absorpColor`. Calculate lit color:
    $$ lit = (shadow \cdot directLight + ambient) \cdot diffAlbedo + (indirectLight \cdot absorpColor) + rim \cdot 0.6 \cdot absorpColor $$
    [S1, unofficial].
12. **Specular Integration:** Compute microfacet specular term using GGX-like distribution [S1, unofficial]:
    *   `alpha = roughness * roughness` [S1, unofficial].
    *   `H = normalize(L - V)` [S1, unofficial].
    *   `dotLH = max(0.0, dot(L, H))`, `dotNH = max(0.0, dot(N, H))`, `dotNL = max(0.0, dot(N, L))` [S1, unofficial].
    *   `alphaSqr = alpha * alpha` [S1, unofficial].
    *   `denom = dotNH * dotNH * (alphaSqr - 1.0) + 1.0` [S1, unofficial].
    *   `D = alphaSqr / (PI * denom * denom + 1e-6)` [S1, unofficial].
    *   `F = F0 + (1.0 - F0) * pow(1.0 - dotLH, 5.0)` [S1, unofficial].
    *   `k = 0.5 * alpha`, `k2 = k * k` [S1, unofficial].
    *   Specular contribution: `spec = shadow * (dotNL * D * F / (max(1e-6, dotLH * dotLH * (1.0 - k2)) + k2)) * absorpColor * lit` [S1, unofficial].
    *   `lit += spec` [S1, unofficial].
    *   Apply shadow to final lit value: `lit *= max(0.7, shadow)` [S1, unofficial].
13. **Tone Mapping:** Apply Reinhard-like tone mapping: `finalColor = 1.0 - exp(-2.0 * lit)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_shading_pipeline(camPos, rayDir, tr, time):
    # tr is TraceResult with hit, t, cell, normal
    if not tr.hit:
        return sky_color # Computed via atmosphere or absorpColor

    # 1. Hit Position
    if tr.t <= 0.0:
        hitPosition = camPos
    else:
        hitPosition = camPos + rayDir * (tr.t + 1e-5)
    
    hitCell = tr.cell
    hitFaceNormal = tr.normal
    N = normalize(hitFaceNormal)
    V = normalize(rayDir)
    L = normalize(sun_vec(time * SUN_ROT_SPEED))

    # 2. Atmosphere/Sky
    absorpColor = compute_absorption(vec3(0,1,0), camPos, L, ...) # Implementation details omitted
    if ENABLE_ATMOSPHERE:
        sky = 1.0 - exp(atmosphere(rayDir, camPos, L, ...))
    else:
        sky = absorpColor

    # 3. Local Coords & Albedo
    local = hitPosition - vec3(float(hitCell.x), float(hitCell.y), float(hitCell.z))
    localPos = fract(local)
    b = get_block_type(hitCell)
    albedo = sample_block_albedo(hitCell, localPos, N)

    # 4. Basic Lighting Terms
    nDotL = max(0.0, dot(N, L))
    lambert = nDotL
    rim = pow(1.0 - max(0.0, dot(N, V)), 2.2) * 0.12

    lightColor = vec3(1.0, 0.96, 0.85)
    ambient = vec3(0.12, 0.14, 0.18)

    # 5. Shadow
    shadow = 1.0
    if ENABLE_SHADOW:
        sh = trace_voxel(hitPosition + N * 0.01, L, tLimit, maxSteps)
        if sh.hit:
            shadow = 0.0

    # 6. Direct/Indirect Light
    directLight = lightColor * lambert
    indirectLight = 0.25 * albedo

    if b == BLOCK_EMISSIVE:
        emission = get_emissive_color(hitCell) * get_emissive_intensity(hitCell)
        indirectLight += 0.35 * emission
        directLight += emission * 0.5

    # 7. Integration
    diffAlbedo = albedo * absorpColor
    lit = (shadow * directLight + ambient) * diffAlbedo + (indirectLight * absorpColor) + rim * 0.6 * absorpColor

    # 8. Specular (GGX-like)
    roughness = 0.54
    F0 = 0.14
    alpha = roughness * roughness
    H = normalize(L - V)
    dotLH = max(0.0, dot(L, H))
    dotNH = max(0.0, dot(N, H))
    dotNL = max(0.0, dot(N, L))
    
    alphaSqr = alpha * alpha
    denom = dotNH * dotNH * (alphaSqr - 1.0) + 1.0
    D = alphaSqr / (PI * denom * denom + 1e-6)
    F = F0 + (1.0 - F0) * pow(1.0 - dotLH, 5.0)
    k = 0.5 * alpha
    k2 = k * k
    
    spec = shadow * (dotNL * D * F / (max(1e-6, dotLH * dotLH * (1.0 - k2)) + k2)) * absorpColor * lit
    lit += spec
    lit *= max(0.7, shadow)

    # 9. Tone Mapping
    finalColor = 1.0 - exp(-2.0 * lit)
    return finalColor
```

**Edge Cases and Pitfalls:**
*   **Precision:** Use `1e-5` offset in ray origin to avoid self-intersection artifacts [S1, unofficial].
*   **Shadow Bias:** Use `N * 0.01` offset for shadow ray origin [S1, unofficial].
*   **Division by Zero:** Add `1e-6` to denominators in specular calculation [S1, unofficial].
*   **Overbright:** Atmosphere sky color is exponentiated negatively to reduce brightness [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** O(1) per pixel for shading, plus O(N) for shadow ray marching where N is steps to sun [S1, unofficial].
*   **Memory Complexity:** O(1) per pixel, plus texture memory for albedo [S1, unofficial].
*   **Performance:** Interactive on modern GPUs due to simple shading model and hard shadows [S2, unofficial].
*   **Strengths:** Simple implementation, visually pleasing for voxel worlds, handles emissive blocks [S1, S2, unofficial].
*   **Limitations:** Hard shadows only, no soft shadows or global illumination [S1, S2, unofficial].
*   **When to use:** Interactive voxel games, real-time rendering [S2, unofficial].
*   **When not to use:** Photorealistic rendering requiring soft shadows or GI [S2, unofficial].

## Background
The method originates from community-driven voxel shader implementations, specifically the "Voxel-World-Shader" by razzle-reinizu [S1, unofficial] and "Ray-Marched Voxel Simulation" by Joris Rijsdijk [S2, unofficial]. It is used in indie voxel games and educational graphics projects [S1, S2, unofficial].

## Sources
[S1] GitHub - razzle-reinizu/Voxel-World-Shader: A GLSL voxel-world shader ..., https://github.com/razzle-reinizu/Voxel-World-Shader, fetched 2026-10-25 (unofficial)
[S2] Ray-Marched Voxel Simulation - Joris Rijsdijk, https://www.jorisar.nl/blog/VoxelPlayground/, fetched 2026-10-25 (unofficial)
