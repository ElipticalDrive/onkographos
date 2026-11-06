# Voxel Shading Pipeline

Also called: Block Shading, Shading and Lighting, Voxel Lighting

## Overview
The Voxel Shading Pipeline is a per-voxel lighting model that computes illumination for surface points identified by ray marching through a voxel grid [S1, S2, unofficial]. It combines Lambertian diffuse reflection, rim lighting, indirect ambient contribution, and a microfacet-based specular term, modulated by hard shadows and tone mapping [S1, unofficial]. It differs from continuous surface shading by operating on discrete voxel coordinates, often using the voxel face normal directly rather than interpolating across a mesh, and typically employs hard shadows due to the binary nature of voxel occupancy [S2, unofficial].

## How it works
**Inputs:**
- Camera position $\mathbf{c}$ and ray direction $\mathbf{v}_{ray}$ [S1, unofficial].
- Voxel grid data: block type, albedo texture, emissive intensity [S1, unofficial].
- Light direction $\mathbf{L}$ (sun vector) [S1, S2, unofficial].
- Parameters: `lightColor`, `ambient`, `roughness`, `F0`, `tLimit`, `maxSteps` [S1, unofficial].

**Outputs:**
- Final pixel color $\mathbf{C}_{final}$ [S1, unofficial].

**Data Structures:**
- `TraceResult`: Contains `hit` (bool), `t` (float, distance), `cell` (ivec3, voxel coordinates), `normal` (vec3, face normal) [S1, unofficial].
- `BlockType`: Enum identifying material properties (e.g., `BLOCK_EMISSIVE`) [S1, unofficial].

**Steps:**

1.  **Ray Marching:** Trace the ray from the camera. If a hit occurs at distance $t$, compute the hit position $\mathbf{p}$ and the voxel cell $\mathbf{c}_{vox}$ [S1, unofficial].
    $$ \mathbf{p} = \begin{cases} \mathbf{c} & \text{if } t \le 0 \\ \mathbf{c} + \mathbf{v}_{ray} (t + \epsilon) & \text{otherwise} \end{cases} $$
    where $\epsilon = 10^{-5}$ [S1, unofficial].

2.  **Geometric Setup:**
    - Compute the surface normal $\mathbf{n}$ from the hit face normal [S1, unofficial].
    - Compute the view direction $\mathbf{V} = -\mathbf{v}_{ray}$ (normalized) [S1, unofficial].
    - Compute the light direction $\mathbf{L}$ (normalized) [S1, S2, unofficial].
    - Calculate the local position within the voxel: $\mathbf{p}_{local} = \text{fract}(\mathbf{p} - \mathbf{c}_{vox})$ [S1, unofficial].

3.  **Material Sampling:**
    - Sample the albedo $\mathbf{A}$ using the voxel cell and local position [S1, unofficial].
    - Determine block type $b$ [S1, unofficial].

4.  **Lighting Components:**
    - **Diffuse:** $\mathbf{D}_{diff} = \max(0, \mathbf{n} \cdot \mathbf{L})$ [S1, unofficial].
    - **Rim:** $\mathbf{R} = \text{pow}(1.0 - \max(0, \mathbf{n} \cdot \mathbf{V}), 2.2) \times 0.12$ [S1, unofficial].
    - **Shadow:** Cast a secondary ray from $\mathbf{p} + \mathbf{n}\epsilon$ in direction $\mathbf{L}$. If it hits, shadow factor $S = 0$, else $S = 1$ [S1, S2, unofficial].
    - **Direct Light:** $\mathbf{L}_{dir} = \mathbf{C}_{light} \times \mathbf{D}_{diff}$ [S1, unofficial].
    - **Indirect Light:** $\mathbf{L}_{ind} = 0.25 \times \mathbf{A}$ [S1, unofficial].
    - **Emissive:** If $b = \text{BLOCK\_EMISSIVE}$, add emission $\mathbf{E}$:
      $$ \mathbf{L}_{ind} \leftarrow \mathbf{L}_{ind} + 0.35 \mathbf{E} $$
      $$ \mathbf{L}_{dir} \leftarrow \mathbf{L}_{dir} + 0.5 \mathbf{E} $$
      [S1, unofficial].

5.  **Accumulation:**
    Combine terms using the absorption color $\mathbf{C}_{abs}$ (from atmosphere or base color) [S1, unofficial]:
    $$ \mathbf{L}_{lit} = (S \cdot \mathbf{L}_{dir} + \mathbf{C}_{amb}) \cdot (\mathbf{A} \cdot \mathbf{C}_{abs}) + (\mathbf{L}_{ind} \cdot \mathbf{C}_{abs}) + \mathbf{R} \cdot 0.6 \cdot \mathbf{C}_{abs} $$
    [S1, unofficial].

6.  **Specular (Microfacet):**
    Compute a GGX-like specular term [S1, unofficial]:
    - Half vector $\mathbf{H} = \text{normalize}(\mathbf{L} + \mathbf{V})$ [S1, unofficial].
    - $\alpha = \text{roughness}^2$ [S1, unofficial].
    - Normal Distribution $D = \frac{\alpha^2}{\pi (\cos^2\theta_h (\alpha^2 - 1) + 1)^2}$ [S1, unofficial].
    - Fresnel $F = F_0 + (1 - F_0)(1 - \mathbf{L} \cdot \mathbf{H})^5$ [S1, unofficial].
    - Geometry/Approximation factor $k = 0.5\alpha$ [S1, unofficial].
    - Add to lit color:
      $$ \mathbf{L}_{lit} \leftarrow \mathbf{L}_{lit} + S \cdot \frac{(\mathbf{n} \cdot \mathbf{L}) D F}{\max(\epsilon, (\mathbf{L} \cdot \mathbf{H})^2 (1 - k^2)) + k^2} \cdot \mathbf{C}_{abs} \cdot \mathbf{L}_{lit} $$
      [S1, unofficial].
    - Modulate by shadow: $\mathbf{L}_{lit} \leftarrow \mathbf{L}_{lit} \cdot \max(0.7, S)$ [S1, unofficial].

7.  **Tone Mapping:**
    $$ \mathbf{C}_{final} = 1.0 - \exp(-2.0 \cdot \mathbf{L}_{lit}) $$
    [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_shading_pipeline(ray_dir, cam_pos, sun_vec, time, grid, params):
    # 1. Ray Marching
    tr = trace_voxel(cam_pos, ray_dir, params.tLimit, params.maxSteps)
    if not tr.hit:
        return sky_color(ray_dir, sun_vec)

    # Hit position calculation
    if tr.t <= 0.0:
        hit_pos = cam_pos
    else:
        hit_pos = cam_pos + ray_dir * (tr.t + 1e-5)
    
    hit_cell = tr.cell
    hit_normal = tr.normal
    
    # 2. Geometric Setup
    N = normalize(hit_normal)
    L = normalize(sun_vec(time))
    V = normalize(-ray_dir)
    
    # Local position for texture sampling
    local_pos = fract(hit_pos - vec3(float(hit_cell.x), float(hit_cell.y), float(hit_cell.z)))
    
    # 3. Material Sampling
    block_type = get_block_type(hit_cell)
    albedo = sample_albedo(hit_cell, local_pos)
    
    # 4. Lighting Components
    n_dot_l = max(0.0, dot(N, L))
    lambert = n_dot_l
    rim = pow(1.0 - max(0.0, dot(N, V)), 2.2) * 0.12
    
    light_color = vec3(1.0, 0.96, 0.85)
    ambient = vec3(0.12, 0.14, 0.18)
    
    # Hard Shadow
    shadow = 1.0
    if enable_shadow:
        sh = trace_voxel(hit_pos + N * 0.01, L, params.tLimit, params.maxSteps)
        if sh.hit:
            shadow = 0.0
            
    direct_light = light_color * lambert
    indirect_light = 0.25 * albedo
    
    # Emissive Handling
    if block_type == BLOCK_EMISSIVE:
        emission = get_emissive_color(hit_cell) * get_emissive_intensity(hit_cell)
        indirect_light += 0.35 * emission
        direct_light += 0.5 * emission
        
    # Absorption/Atmosphere Color
    absorp_color = compute_absorption(cam_pos, L)
    
    # 5. Accumulation
    diff_albedo = albedo * absorp_color
    lit = (shadow * direct_light + ambient) * diff_albedo + (indirect_light * absorp_color) + rim * 0.6 * absorp_color
    
    # 6. Specular (GGX-like)
    roughness = 0.54
    F0 = 0.14
    alpha = roughness * roughness
    alpha_sqr = alpha * alpha
    
    H = normalize(L + V)
    dot_LH = max(0.0, dot(L, H))
    dot_NH = max(0.0, dot(N, H))
    dot_NL = max(0.0, dot(N, L))
    
    denom = dot_NH * dot_NH * (alpha_sqr - 1.0) + 1.0
    D = alpha_sqr / (PI * denom * denom + 1e-6)
    F = F0 + (1.0 - F0) * pow(1.0 - dot_LH, 5.0)
    k = 0.5 * alpha
    k2 = k * k
    
    specular_term = shadow * (dot_NL * D * F / (max(1e-6, dot_LH * dot_LH * (1.0 - k2)) + k2))
    lit += specular_term * absorp_color * lit
    
    lit *= max(0.7, shadow)
    
    # 7. Tone Mapping
    final_color = 1.0 - exp(-2.0 * lit)
    
    return final_color
```

**Parameters:**
- `lightColor`: RGB $(1.0, 0.96, 0.85)$ [S1, unofficial].
- `ambient`: RGB $(0.12, 0.14, 0.18)$ [S1, unofficial].
- `roughness`: $0.54$ [S1, unofficial].
- `F0`: $0.14$ [S1, unofficial].

**Edge Cases:**
- If $t \le 0$, the hit position is set to the camera position [S1, unofficial].
- Precision issues in denominator of specular term are guarded with $\epsilon = 10^{-6}$ [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of steps in ray marching, plus one additional trace for shadowing if enabled [S1, S2, unofficial].
- **Memory Complexity:** $O(1)$ per pixel, assuming the voxel grid is accessible via hashing or uniform sampling [S1, unofficial].
- **Strengths:** Simple to implement; hard shadows are cheap in voxel grids; fits the aesthetic of block-based worlds [S2, unofficial].
- **Limitations:** Hard shadows can appear jagged; specular highlights are per-voxel, potentially causing aliasing at grazing angles [S2, unofficial].

## Background
The method originates from interactive voxel rendering techniques popularized in indie game development and educational ray marching projects [S1, S2, unofficial]. It adapts standard PBR concepts (GGX, Fresnel) to the discrete voxel domain, often simplifying geometry to face normals [S1, S2, unofficial].

## Sources
[S1] GitHub - razzle-reinizu/Voxel-World-Shader: A GLSL voxel-world shader, https://github.com/razzle-reinizu/Voxel-World-Shader, fetched 2026-10-25 (unofficial)
[S2] Ray-Marched Voxel Simulation - Joris Rijsdijk, https://www.jorisar.nl/blog/VoxelPlayground/, fetched 2026-10-25 (unofficial)
