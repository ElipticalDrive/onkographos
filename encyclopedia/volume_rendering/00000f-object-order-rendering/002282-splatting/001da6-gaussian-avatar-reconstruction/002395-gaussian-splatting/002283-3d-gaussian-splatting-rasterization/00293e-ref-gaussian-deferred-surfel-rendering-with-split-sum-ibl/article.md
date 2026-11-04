# Ref-Gaussian Deferred Surfel Rendering with Split-Sum IBL

Also called: MetalGaussianSplatRelighting, Ref-Gaussian on iOS

## Overview
Ref-Gaussian Deferred Surfel Rendering with Split-Sum IBL is a real-time deferred shading method for relighting 3D Gaussian Splatting (3DGS) models on mobile hardware (specifically iOS/Metal) [S1, unofficial]. It solves the problem of dynamic environment lighting in Gaussian splats by decomposing splats into physically-based rendering (PBR) materials (albedo, normal, roughness, reflectance) and computing lighting via a deferred pipeline with split-sum environment map integration [S1, unofficial]. Unlike forward-shading Gaussian splatters that bake lighting into spherical harmonics or use analytic approximations, this method separates the material definition from the lighting environment, allowing the environment map to be swapped or rotated in real-time without retraining or re-rasterizing the splats [S1, unofficial]. It differs from standard PBR deferred renderers by adapting the splat-based rasterization and surfel accumulation to the deferred G-buffer workflow [S1, unofficial].

## How it works
**Inputs:**
- A `.ply` file containing 3D Gaussian splats with per-splat attributes: position, covariance (or quaternion/scale), opacity, spherical harmonics (SH) coefficients for view-dependent color, and new material attributes: normal, roughness, reflectance (specular strength), and albedo (diffuse color) [S1, unofficial].
- An HDR environment map (equirectangular or cubemap) representing the lighting [S1, unofficial].

**Outputs:**
- A final color buffer with relit splats [S1, unofficial].

**Data Structures:**
- **Splat Buffer:** Contains position, rotation (quaternion), scale, opacity, and view-dependent SH coefficients. Sorted for cache locality (e.g., Morton code) [S1, unofficial].
- **Material Buffer:** Parallel array to the Splat Buffer, containing per-splat material properties: normal (3 floats), roughness (1 float), reflectance (1 float), and albedo (3 floats). Must be sorted in parallel with the Splat Buffer to maintain alignment [S1, unofficial].
- **G-Buffer:** Tile memory or render target storing per-pixel accumulated material data (normal, roughness, reflectance, albedo) and depth [S1, unofficial].
- **IBL Pre-computed Textures:**
  - Prefiltered Environment Map: Mipmapped cubemap/equirect where each mip level corresponds to a specific roughness range [S1, unofficial].
  - Irradiance Map: Pre-filtered environment map for diffuse lighting (not used for diffuse color in this method, but part of the standard split-sum setup) [S1, unofficial].
  - BRDF LUT: 2D texture mapping $(F_0, \text{roughness})$ to $(fg.x, fg.y)$ coefficients for the Schlick-Beckmann approximation [S1, unofficial].

**Steps:**

1. **Load and Parse:** Read the `.ply` file. Extract per-splat geometric data (position, quaternion, scale) and material data (normal, roughness, reflectance, albedo) [S1, unofficial].
2. **Sort for Locality:** Sort the Splat Buffer using a spatial locality algorithm (e.g., Morton code) to optimize cache performance during rasterization [S1, unofficial].
3. **Synchronize Material Buffer:** Apply the exact same sort indices to the Material Buffer to ensure each splat's geometry aligns with its material properties [S1, unofficial].
4. **Pre-compute IBL:** Use a compute kernel to generate:
   - The Prefiltered Environment Map by convolving the HDR environment map with a microfacet distribution for various roughness levels [S1, unofficial].
   - The Irradiance Map by convolving the HDR environment map with a hemisphere integral (not used for diffuse color in this specific implementation) [S1, unofficial].
   - The BRDF LUT by integrating the Fresnel and geometry terms [S1, unofficial].
5. **Rasterize to G-Buffer:**
   - Project splats to screen space.
   - For each pixel covered by a splat, accumulate the material properties (normal, roughness, reflectance, albedo) and depth into the G-Buffer. Standard alpha blending or weighted accumulation is used for the material attributes [S1, unofficial].
6. **Deferred Shading (Split-Sum IBL):**
   - For each pixel in the G-Buffer, retrieve the accumulated material properties $(N, \text{roughness}, \text{reflectance}, \text{albedo})$ and the view direction $V$ [S1, unofficial].
   - Compute the base specular reflectivity $F_0$ [S1, unofficial]:
     $$ F_0 = 0.04 \times (1 - \text{reflectance}) + \text{albedo} \times \text{reflectance} $$
     where $F_0$ is the base specular reflectivity, $\text{reflectance}$ is the learned specular strength, and $\text{albedo}$ is the diffuse color [S1, unofficial].
   - Compute the specular term [S1, unofficial]:
     $$ \text{specular} = \text{prefiltered}(\text{reflect}(V, N), \text{roughness}) \times (F_0 \cdot fg.x + fg.y) $$
     where $\text{prefiltered}(\cdot, \cdot)$ samples the prefiltered environment map at the reflection direction $\text{reflect}(V, N)$ using the mip level determined by $\text{roughness}$, and $fg$ is the result from the BRDF LUT sampled with $(F_0, \text{roughness})$ [S1, unofficial].
   - Compute the final color [S1, unofficial]:
     $$ \text{final} = (1 - \text{reflectance}) \times \text{base\_color} + \text{specular} $$
     where $\text{base\_color}$ is the albedo used directly for the diffuse component without irradiance multiplication [S1, unofficial].
7. **Post-processing:** Composite the final color with the skybox or background environment map if necessary [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1-3: Data Loading and Sorting
splat_buffer = load_ply_splats("model.ply") # Contains pos, quat, scale, opacity, sh
material_buffer = load_ply_materials("model.ply") # Contains normal, roughness, reflectance, albedo

# Sort splats for cache locality (e.g., Morton code)
sorted_indices = compute_morton_sort(splat_buffer.positions)
splat_buffer.sort_in_place(sorted_indices)
material_buffer.sort_in_place(sorted_indices) # Critical: keep materials aligned

# Step 4: IBL Pre-computation (Compute Kernel)
prefiltered_env_map = prefilter_environment_map(hdr_env_map, roughness_levels)
brdf_lut = compute_brdf_lut()
# irradiance_map is computed but not used for diffuse color in this method

# Step 5: Rasterization to G-Buffer
g_buffer = initialize_g_buffer(width, height)
for splat in splat_buffer:
    screen_rect = project_to_screen(splat.pos, splat.quat, splat.scale, camera_matrix)
    for pixel in screen_rect:
        if pixel inside splat coverage:
            material = material_buffer[splat.index]
            # Accumulate material properties into G-Buffer
            # Note: Normal accumulation may require normalization after blending
            g_buffer[pixel].normal += material.normal
            g_buffer[pixel].roughness += material.roughness
            g_buffer[pixel].reflectance += material.reflectance
            g_buffer[pixel].albedo += material.albedo
            g_buffer[pixel].depth = min(g_buffer[pixel].depth, splat.depth)

# Step 6: Deferred Shading
for pixel in g_buffer:
    N = normalize(g_buffer[pixel].normal)
    roughness = g_buffer[pixel].roughness
    reflectance = g_buffer[pixel].reflectance
    albedo = g_buffer[pixel].albedo
    V = get_view_direction(pixel, camera_matrix)
    
    # Compute F0
    F0 = 0.04 * (1.0 - reflectance) + albedo * reflectance
    
    # Compute Specular
    R = reflect(V, N)
    # Sample prefiltered env map using roughness to select mip level
    prefiltered_color = sample_prefiltered_env_map(prefiltered_env_map, R, roughness)
    # Sample BRDF LUT
    fg = sample_brdf_lut(brdf_lut, F0, roughness)
    specular = prefiltered_color * (F0 * fg.x + fg.y)
    
    # Compute Final Color
    # Diffuse is just base_color (albedo), no irradiance multiplication
    final_color = (1.0 - reflectance) * albedo + specular
    
    output_buffer[pixel] = final_color
```

**Parameters:**
- $F_0$ base value: 0.04 for dielectrics [S1, unofficial].
- Roughness: Controls specular blur. Low roughness (~0.2 for cars) results in blurry reflections; high roughness results in diffuse-like specular [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Normal Map Noise:** If splats are sorted for cache locality but the material buffer is not sorted in parallel, normals become misaligned with positions, causing rainbow noise [S1, unofficial]. Fix: Apply the same sort indices to the material buffer [S1, unofficial].
- **Incorrect Diffuse Lighting:** Do not multiply albedo by the irradiance map for diffuse color. Ref-Gaussian uses base_color directly for diffuse. Only specular is computed from the environment map [S1, unofficial].
- **Coordinate System Mismatch:** If scene data is Z-up (e.g., Blender) but the viewer assumes Y-up, objects appear rotated 90 degrees. Fix: Apply a -90° rotation around the X-axis to the camera/view matrix and adjust environment map sampling directions to match the scene's up-axis [S1, unofficial].
- **Stale Binary Execution on iOS:** If iOS build fails due to type errors (e.g., Float16 bitPattern type mismatch between arm64 and x86_64 simulator), the device may run an old binary. Always check build success for the target platform before debugging logic [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Rasterization is $O(N)$ where $N$ is the number of splats, with overhead for sorting and G-buffer accumulation. Shading is $O(P)$ where $P$ is the number of pixels, with constant-time texture lookups for IBL.
- **Memory Complexity:** $O(N)$ for splat and material buffers, $O(P)$ for G-buffer, and constant space for IBL pre-computed textures (size depends on environment map resolution).
- **Measured Performance:** Real-time on iPhone using Metal [S1, unofficial].
- **Scaling:** Scales with the number of splats and screen resolution. IBL pre-computation is done once or when the environment map changes.
- **Strengths:** Real-time relighting, physically-based materials, mobile-friendly (Metal).
- **Limitations:** Requires material attributes (normal, roughness, etc.) to be learned or provided in the `.ply` file. Deferred shading adds memory overhead for the G-buffer.
- **When to use:** When dynamic environment lighting is required for 3DGS models on mobile devices.
- **When not to use:** When forward shading is preferred for simplicity or when material attributes are not available.

## Background
Ref-Gaussian Deferred Surfel Rendering with Split-Sum IBL was developed by Daisuke Majima (MLBoy) and published on Qiita in May 2026 [S1, unofficial]. It is an implementation of the Ref-Gaussian method, which extends 3D Gaussian Splatting with PBR materials and deferred shading for real-time relighting on iOS [S1, unofficial]. The method builds on the split-sum approximation for environment lighting introduced in Unreal Engine 4 [S1, unofficial].

## Sources
[S1] iPhoneで Gaussian Splatting モデルの反射をリアルタイムで更新する, https://qiita.com/john-rocky/items/3c452476e66bdc29c290, fetched 2026-10-29 (unofficial)
