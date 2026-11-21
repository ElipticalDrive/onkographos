# Voxel Shading Pipeline

Also called: Block Shading, Shading and Lighting, Voxel Lighting

## Overview
The Voxel Shading Pipeline computes the final color of a voxel hit during ray marching by combining albedo sampling, Lambertian diffuse lighting, rim lighting, and specular integration [S1, unofficial]. It distinguishes itself from continuous shading by operating on discrete voxel coordinates and neighbor-dependent lighting calculations, specifically using hard shadow rays and per-voxel specular highlights rather than smooth surface approximations [S2, unofficial]. It serves as the foundational lighting model for voxel-specific rendering, upon which more complex effects like refraction or global illumination are built [S1, unofficial].

## How it works
**Inputs:**
- `hitPosition`: World-space position of the ray-voxel intersection.
- `hitCell`: Integer coordinates $(x, y, z)$ of the hit voxel.
- `hitFaceNormal`: Normal vector of the hit voxel face.
- `rayDir`: Camera ray direction.
- `sunVec`: World-space sun direction.
- `time`: Current simulation time (for sun rotation).
- `tLimit`, `maxSteps`: Parameters for shadow ray tracing.

**Outputs:**
- `finalColor`: The computed RGB color of the pixel.

**Data Structures:**
- `TraceResult`: Struct containing `hit` (bool), `t` (float, distance), `cell` (ivec3), `normal` (vec3).
- `BLOCK_EMISSIVE`: Integer constant identifying emissive block types.

**Steps:**

1.  **Intersection Validation:**
    If the ray hits a voxel (`tr.hit` is true):
    - If `tr.t <= 0.0`, set `hitPosition = camPos`.
    - Else, set `hitPosition = camPos + rayDir * (tr.t + 1e-5)` [S1, unofficial].
    - Extract `hitCell = tr.cell` and `hitFaceNormal = tr.normal` [S1, unofficial].

2.  **Vector Normalization:**
    - $N = \text{normalize}(\text{hitFaceNormal})$
    - $L = \text{normalize}(\text{sunVec}(time \times \text{SUN\_ROT\_SPEED}))$
    - $V = \text{normalize}(\text{rayDir})$ [S1, unofficial]

3.  **Local Coordinate Calculation:**
    - Compute local position within the voxel:
      $$ \text{local} = \text{hitPosition} - \text{vec3}(\text{float}(\text{hitCell}.x), \text{float}(\text{hitCell}.y), \text{float}(\text{hitCell}.z)) $$
    - Apply fracturing for texture sampling:
      $$ \text{localPos} = \text{fract}(\text{local}) $$ [S1, unofficial]

4.  **Albedo and Environment Sampling:**
    - Sample block albedo: `albedo = sampleBlockAlbedoTex_manual(hitCell, localPos, hitFaceNormal)` [S1, unofficial].
    - Compute absorption color: `absorpColor = absorp(...)` [S1, unofficial].
    - Compute sky color:
      - If atmosphere enabled: `sky = 1.0 - exp(-atmosphere(...))`
      - Else: `sky = absorpColor` [S1, unofficial]

5.  **Lighting Components:**
    - **Diffuse:**
      $$ nDotL = \max(0.0, N \cdot L) $$
      $$ \text{lambert} = nDotL $$ [S1, unofficial]
    - **Rim Lighting:**
      $$ \text{rim} = \text{pow}(1.0 - \max(0.0, N \cdot V), 2.2) \times 0.12 $$ [S1, unofficial]
    - **Constants:**
      - `lightColor = vec3(1.0, 0.96, 0.85)`
      - `ambient = vec3(0.12, 0.14, 0.18)` [S1, unofficial]

6.  **Shadow Casting:**
    - Initialize `shadow = 1.0`.
    - If shadows enabled:
      - Trace voxel from `hitPosition + N * 0.01` in direction `L`.
      - If `sh.hit` is true, `shadow = 0.0` [S1, unofficial].

7.  **Direct and Indirect Light:**
    - `directLight = lightColor * lambert`
    - `indirectLight = 0.25 * albedo` [S1, unofficial]
    - **Emissive Handling:**
      - If block type is `BLOCK_EMISSIVE`:
        - `emission = getEmissiveColor(hitCell) * getEmissiveIntensity(hitCell)`
        - `indirectLight += 0.35 * emission`
        - `directLight += emission * 0.5` [S1, unofficial]

8.  **Final Color Composition:**
    - `diffAlbedo = albedo * absorpColor`
    - Base lit color:
      $$ \text{lit} = (\text{shadow} \times \text{directLight} + \text{ambient}) \times \text{diffAlbedo} + (\text{indirectLight} \times \text{absorpColor}) + \text{rim} \times 0.6 \times \text{absorpColor} $$ [S1, unofficial]

9.  **Specular Integration (Microfacet/GGX-like):**
    - Parameters: `roughness = 0.54`, `F0 = 0.14` [S1, unofficial].
    - Compute half vector: $H = \text{normalize}(L - V)$
    - Compute dot products:
      - $dotLH = \max(0.0, L \cdot H)$
      - $dotNH = \max(0.0, N \cdot H)$
      - $dotNL = \max(0.0, N \cdot L)$
    - Compute D (Normal Distribution):
      $$ \alpha = \text{roughness}^2 $$
      $$ \alphaSqr = \alpha^2 $$
      $$ \text{denom} = dotNH^2 \times (\alphaSqr - 1.0) + 1.0 $$
      $$ D = \frac{\alphaSqr}{\pi \times \text{denom}^2 + 1e-6} $$
    - Compute F (Fresnel):
      $$ F = F0 + (1.0 - F0) \times \text{pow}(1.0 - dotLH, 5.0) $$
    - Compute Geometry/Approximation term:
      $$ k = 0.5 \times \alpha $$
      $$ k2 = k^2 $$
      $$ \text{specularTerm} = \frac{dotNL \times D \times F}{\max(1e-6, dotLH^2 \times (1.0 - k2)) + k2} $$
    - Add to lit:
      $$ \text{lit} += \text{shadow} \times \text{specularTerm} \times \text{absorpColor} \times \text{lit} $$ [S1, unofficial]
    - Apply shadow modifier to final lit:
      $$ \text{lit} *= \max(0.7, \text{shadow}) $$ [S1, unofficial]

10. **Tone Mapping:**
    - `finalColor = 1.0 - exp(-2.0 * lit)` [S1, unofficial]

11. **Miss Case:**
    - If no hit, `finalColor = sky` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_shading_pipeline(hit, cam_pos, ray_dir, time, sun_rot_speed, t_limit, max_steps):
    if not hit:
        return compute_sky_color(ray_dir)

    # 1. Intersection Position
    if hit.t <= 0.0:
        hit_position = cam_pos
    else:
        hit_position = cam_pos + ray_dir * (hit.t + 1e-5)
    
    hit_cell = hit.cell
    hit_normal = hit.normal
    
    # 2. Vectors
    N = normalize(hit_normal)
    L = normalize(sun_vec(time * sun_rot_speed))
    V = normalize(ray_dir)
    
    # 3. Local Coords
    local = hit_position - vec3(float(hit_cell.x), float(hit_cell.y), float(hit_cell.z))
    local_pos = fract(local)
    
    # 4. Sampling
    block_type = get_block_type(hit_cell)
    albedo = sample_block_albedo(hit_cell, local_pos, hit_normal)
    absorp_color = compute_absorption(hit_cell, cam_pos, L)
    
    # Sky
    if atmosphere_enabled:
        sky = 1.0 - exp(-compute_atmosphere(ray_dir, cam_pos, L))
    else:
        sky = absorp_color
        
    # 5. Lighting Basics
    n_dot_l = max(0.0, dot(N, L))
    rim = pow(1.0 - max(0.0, dot(N, V)), 2.2) * 0.12
    
    light_color = vec3(1.0, 0.96, 0.85)
    ambient = vec3(0.12, 0.14, 0.18)
    
    # 6. Shadows
    shadow = 1.0
    if shadows_enabled:
        sh = trace_voxel(hit_position + N * 0.01, L, t_limit, max_steps)
        if sh.hit:
            shadow = 0.0
            
    # 7. Direct/Indirect
    direct_light = light_color * n_dot_l
    indirect_light = 0.25 * albedo
    
    if block_type == BLOCK_EMISSIVE:
        emission = get_emissive_color(hit_cell) * get_emissive_intensity(hit_cell)
        indirect_light += 0.35 * emission
        direct_light += emission * 0.5
        
    # 8. Composition
    diff_albedo = albedo * absorp_color
    lit = (shadow * direct_light + ambient) * diff_albedo + (indirect_light * absorp_color) + rim * 0.6 * absorp_color
    
    # 9. Specular
    roughness = 0.54
    F0 = 0.14
    alpha = roughness * roughness
    alpha_sqr = alpha * alpha
    
    H = normalize(L - V)
    dot_LH = max(0.0, dot(L, H))
    dot_NH = max(0.0, dot(N, H))
    dot_NL = max(0.0, dot(N, L))
    
    denom = dot_NH * dot_NH * (alpha_sqr - 1.0) + 1.0
    D = alpha_sqr / (PI * denom * denom + 1e-6)
    F = F0 + (1.0 - F0) * pow(1.0 - dot_LH, 5.0)
    
    k = 0.5 * alpha
    k2 = k * k
    
    spec_term = (dot_NL * D * F) / (max(1e-6, dot_LH * dot_LH * (1.0 - k2)) + k2)
    lit += shadow * spec_term * absorp_color * lit
    
    lit *= max(0.7, shadow)
    
    # 10. Tone Mapping
    final_color = 1.0 - exp(-2.0 * lit)
    
    return final_color
```

## Subtopics

### [Shading with Shadows, Specular, and Refraction](16940-shading-with-shadows-specular-and-refraction.md)
This method extends the basic pipeline by adding shadow ray casting, Phong-like specular highlights, and recursive or discrete refraction bounces. It is distinguished by its support for recursive light transport through semi-transparent volumes and specific handling of voxel-space coordinate offsets for shadow and refraction ray origins.

### Per-Block Lighting with Per-Pixel Adjustment
A lighting optimization that computes illumination per voxel block rather than per pixel, then applies per-pixel modifiers to retain shading detail [S3, unofficial]. It calculates base lighting values for an entire voxel block and applies per-pixel adjustments to prevent uniform shading and maintain surface variation [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Dominated by the shadow ray trace, which requires an additional voxel traversal per lit pixel. The specular calculation involves several trigonometric and power operations per pixel.
- **Memory Complexity:** Low; requires only the voxel grid and albedo textures. No large buffers for global illumination are needed.
- **Strengths:** Fast enough for interactive frame rates on simple scenes; hard shadows and per-voxel speculars provide strong visual cues for voxel structure [S2, unofficial].
- **Limitations:** Hard shadows can appear aliased; no indirect lighting bounce (except simple emissive contribution); specular highlights are discrete per voxel, which may look blocky at high resolutions [S2, unofficial].
- **When to use:** Real-time voxel games where performance is critical and stylized, blocky lighting is acceptable [S2, unofficial].
- **When not to use:** Photorealistic rendering requiring soft shadows or global illumination.

## Background
The method originates from early interactive voxel world shaders, specifically those implementing ray-marched voxel rendering with basic lighting models [S1, S2, unofficial]. It was developed to provide visually distinct lighting for voxel-based environments without the computational cost of full path tracing [S2, unofficial]. The specific implementation details, such as the GGX-like specular approximation and tone mapping, were popularized in community-driven GLSL shader repositories [S1, unofficial].

## Sources
[S1] GitHub - razzle-reinizu/Voxel-World-Shader: A GLSL voxel-world shader ..., https://github.com/razzle-reinizu/Voxel-World-Shader, fetched 2026-10-25 (unofficial)
[S2] Ray-Marched Voxel Simulation - Joris Rijsdijk, https://www.jorisar.nl/blog/VoxelPlayground/, fetched 2026-10-25 (unofficial)
[S3] Smooth terrain with voxel DDA raymarching, https://www.youtube.com/watch?v=CxoC0E7inUg, fetched 2026-10-25 (unofficial)
