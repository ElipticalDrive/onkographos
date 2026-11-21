# Ref-Gaussian Deferred Surfel Rendering with Split-Sum IBL

Also called: MetalGaussianSplatRelighting, Ref-Gaussian, Ref-Gaussian on iOS

## Overview
Ref-Gaussian is a physically based rendering (PBR) method for novel view synthesis and relighting of reflective objects using 2D Gaussian primitives [S1, S2, unofficial]. It solves the problem of blurred reflective details and inaccurate material representation in standard Gaussian Splatting by decomposing splats into PBR material attributes (albedo, normal, roughness, reflectance) and computing lighting via a deferred shading pipeline with split-sum environment map integration [S1, S2, unofficial]. It is distinguished from its sibling [3D Gaussian Splatting Self-Shadowing (T_light)](4487-3d-gaussian-splatting-self-shadowing-t-light.md) by its focus on material-aware deferred shading and split-sum IBL rather than light-space rasterization for self-shadowing, and from [Light Space Rasterization for T_light](4489-light-space-rasterization-for-t-light.md) by its use of a deferred surfel buffer for cache-locality sorting rather than a dedicated light-space depth buffer [S1, S2, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   A `.ply` file containing 2D Gaussian primitives with per-splat attributes: position, covariance (geometry), normal, roughness, reflectance (specular strength), and albedo (diffuse color) [S1, S2, unofficial].
*   An HDR environment map (equirectangular or cubemap) representing the lighting [S2, unofficial].
*   Camera parameters (view and projection matrices) [S1, S2, unofficial].

**Outputs:**
*   A final color image with physically based reflections and diffuse lighting [S1, S2, unofficial].

### Data Structures

**Splat Buffer:**
*   Layout: Array of structs, sorted by Morton code for cache locality [S2, unofficial].
*   Fields: `position` (vec3), `covariance` (mat3 or 6 floats), `normal` (vec3), `roughness` (float), `reflectance` (float), `albedo` (vec3).
*   Note: The material buffer (containing normal, roughness, reflectance, albedo) must be sorted in parallel with the position/covariance buffer using the same sort indices to prevent desynchronization and rainbow noise [S2, unofficial].

**Material Buffer:**
*   Layout: Parallel array to Splat Buffer, aligned by index [S2, unofficial].
*   Fields: `normal`, `roughness`, `reflectance`, `albedo` [S2, unofficial].

**G-Buffer (Deferred Surfel Buffer):**
*   Layout: Tile-based memory (e.g., Metal tile memory) storing accumulated surfel data [S2, unofficial].
*   Fields: `color` (accumulated), `normal` (accumulated), `depth` (accumulated), material attributes if not passed directly to shading pass [S2, unofficial].

**IBL Pre-computation:**
*   Prefiltered Environment Map: Mipmapped texture where each mip level corresponds to a specific roughness value [S2, unofficial].
*   Irradiance Map: Pre-filtered environment map for diffuse lighting [S2, unofficial].
*   BRDF LUT: 2D texture mapping (roughness, dot(N, V)) to (F0, F) terms [S2, unofficial].

### Numbered Steps

1.  **Loading and Initialization:**
    *   Load the `.ply` file to extract per-splat materials: normal, roughness, reflectance, and albedo [S2, unofficial].
    *   If training, reset all color and material attributes, retaining only the geometry of Gaussians before the deferred rendering stage [S1, unofficial].

2.  **IBL Pre-computation:**
    *   Generate a prefiltered environment map for specular reflections (mipmapped by roughness) [S2, unofficial].
    *   Generate an irradiance map for diffuse reflections [S2, unofficial].
    *   Generate a BRDF LUT [S2, unofficial].

3.  **Sorting:**
    *   Sort splats using spatial locality algorithms like Morton codes to optimize cache usage during rendering [S2, unofficial].
    *   Apply the exact same sort indices to the Material Buffer to ensure normals and other attributes remain aligned with positions [S2, unofficial].

4.  **Rasterization (G-Buffer Pass):**
    *   Project 2D Gaussians to screen space.
    *   Rasterize splats into the G-Buffer, accumulating color, normal, and depth information [S2, unofficial].
    *   Note: During training, material-aware normal propagation is conducted for Gaussians with metallic >= 0.02 and roughness <= 0.1 to capture accurate geometry [S1, unofficial].

5.  **Shading (Split-Sum IBL Pass):**
    *   For each pixel, retrieve surfel data from the G-Buffer.
    *   Compute the base specular reflectivity $F_0$:
        $$ F_0 = 0.04 \times (1 - \text{reflectance}) + \text{albedo} \times \text{reflectance} $$
        where $F_0$ is the base specular reflectivity, reflectance is the learned specular strength, and albedo is the diffuse color [S2, unofficial].
    *   Compute the specular term using the prefiltered environment map and BRDF LUT:
        $$ \text{specular} = \text{prefiltered}(\text{reflect}(V, N), \text{roughness}) \times (F_0 \cdot fg.x + fg.y) $$
        where $V$ is the view direction, $N$ is the normal, and $fg$ is the result from the BRDF LUT [S2, unofficial].
    *   Compute the final color:
        $$ \text{final} = (1 - \text{reflectance}) \times \text{base\_color} + \text{specular} $$
        where `base_color` is the albedo used directly for diffuse lighting without irradiance multiplication [S2, unofficial].
    *   Note: Do not multiply albedo by the irradiance map for diffuse color; Ref-Gaussian uses `base_color` directly for diffuse [S2, unofficial].

6.  **Post-processing:**
    *   Composite the shaded result with the skybox/environment map if necessary [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_ref_gaussian_shading(splat, environment_map, brdf_lut, view_direction):
    """
    Computes the final color for a single surfel using split-sum IBL.
    
    Args:
        splat: Object containing position, normal, roughness, reflectance, albedo.
        environment_map: Object with prefiltered_env_map and irradiance_map.
        brdf_lut: 2D texture for BRDF integration.
        view_direction: Normalized vector from camera to splat.
        
    Returns:
        final_color: vec3
    """
    # Extract material attributes
    N = normalize(splat.normal)
    roughness = splat.roughness
    reflectance = splat.reflectance
    albedo = splat.albedo
    
    # Compute F0 (Base Specular Reflectivity)
    # F0 = 0.04 * (1 - reflectance) + albedo * reflectance
    F0 = 0.04 * (1.0 - reflectance) + albedo * reflectance
    
    # Compute specular term
    # 1. Reflect view direction around normal
    R = reflect(-view_direction, N)
    
    # 2. Sample prefiltered environment map based on roughness
    # The prefiltered map is mipmapped by roughness.
    # roughness maps to a mip level.
    mip_level = roughness_to_mip_level(roughness)
    prefiltered_color = sample_texture(environment_map.prefiltered_env_map, R, mip_level)
    
    # 3. Sample BRDF LUT
    # LUT input: (roughness, dot(N, V))
    # Output: (F0_scale, F_scale) or similar parameters for the Schlick-GGX approximation
    dot_NV = max(dot(N, view_direction), 0.0)
    fg = sample_texture(brdf_lut, vec2(roughness, dot_NV))
    
    # Specular = Prefiltered Env * (F0 * fg.x + fg.y)
    specular = prefiltered_color * (F0 * fg.x + fg.y)
    
    # Compute diffuse term
    # Ref-Gaussian uses base_color directly for diffuse, not albedo * irradiance
    diffuse = albedo
    
    # Final color
    # final = (1 - reflectance) * base_color + specular
    final_color = (1.0 - reflectance) * diffuse + specular
    
    return final_color

def sort_splats_for_cache(splats, materials):
    """
    Sorts splats and materials in parallel using Morton codes.
    
    Args:
        splats: List of splat objects.
        materials: List of material objects aligned with splats.
        
    Returns:
        sorted_splats, sorted_materials
    """
    # Compute Morton codes for splat positions
    morton_codes = [compute_morton_code(s.position) for s in splats]
    
    # Sort indices based on Morton codes
    indices = sorted(range(len(morton_codes)), key=lambda k: morton_codes[k])
    
    # Apply same indices to both buffers
    sorted_splats = [splats[i] for i in indices]
    sorted_materials = [materials[i] for i in indices]
    
    return sorted_splats, sorted_materials
```

### Parameters
*   **Learning rate for material attributes (metallic, roughness, albedo):** 0.005 [S1, unofficial].
*   **Learning rate for environment map:** 0.01 [S1, unofficial].
*   **Balancing weight for RGB reconstruction loss ($\lambda$):** 0.2 [S1, unofficial].
*   **Weight for normal consistency loss ($\lambda_n$):** 0.05 [S1, unofficial].
*   **Weight for normal smoothness loss ($\lambda_{smooth}$):** 1.0 [S1, unofficial].
*   **Initial roughness value:** 0.1 [S1, unofficial].
*   **Initial metallic value:** Adopted from 3DGS-DR (value not explicitly stated) [S1, unofficial].
*   **Roughness:** Controls specular blur. Low roughness (~0.2 for cars) results in blurry reflections; high roughness results in diffuse-like specular [S2, unofficial].
*   **F0 base value:** 0.04 for dielectrics, adjusted by reflectance and albedo [S2, unofficial].

### Edge Cases and Pitfalls
*   **Normal Map Noise:** If splats are sorted (e.g., by Morton code for cache locality) but the material buffer (containing normals) is not sorted in parallel, normals become misaligned with positions, causing rainbow noise. Fix: Apply the same sort indices to the material buffer [S2, unofficial].
*   **Incorrect Diffuse Lighting:** Do not multiply albedo by the irradiance map for diffuse color. Ref-Gaussian uses `base_color` directly for diffuse. Only specular is computed from the environment map [S2, unofficial].
*   **Coordinate System Mismatch:** If the scene data is Z-up (Blender) but the viewer assumes Y-up, objects appear rotated 90 degrees. Fix: Apply a -90° rotation around the X-axis to the camera/view matrix and adjust environment map sampling directions to match the scene's up-axis [S2, unofficial].
*   **Stale Binary Execution on iOS:** If iOS build fails due to type errors (e.g., Float16 bitPattern type mismatch between arm64 and x86_64 simulator), the device may run an old binary. Always check build success for the target platform before debugging logic [S2, unofficial].
*   **Geometric Collapse:** Smooth surfaces are prone to geometric collapse. Material-aware normal propagation addresses this [S1, unofficial].
*   **Blurred Reflective Details:** Per-Gaussian rendering causes reflective details to become noticeably more blurred; deferred rendering is used to mitigate this [S1, unofficial].

## Operating characteristics

**Time and Memory Complexity:**
*   Training consists of two stages: (1) per-Gaussian rendering for 18,000 steps; (2) deferred rendering stage training for about 40,000 steps [S1, unofficial].
*   The deferred rendering pipeline adds overhead for G-Buffer accumulation but improves cache locality and rendering quality [S1, S2, unofficial].

**Measured Performance:**
*   On the Shiny Blender dataset: PSNR 30.33, SSIM 0.958, LPIPS 0.049 [S1, unofficial].
*   MAE 2.15, Normal SSIM 0.954, Normal LPIPS 0.0511, Envmap LPIPS 0.4431 [S1, unofficial].
*   Training time: 0.58 hours [S1, unofficial].
*   FPS: 122 on a single NVIDIA A6000 GPU [S1, unofficial].

**Scaling:**
*   Real-time rendering is supported on mobile devices (iPhone) using Metal [S2, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Physically based rendering with spherical harmonics for diffuse lighting; precise geometric representation using 2D Gaussian primitives; real-time relighting capability [S1, S2, unofficial].
*   **Limitations:** Per-Gaussian shading initialization can cause blurred reflective details if deferred rendering is not used; smooth surfaces are prone to geometric collapse without material-aware normal propagation [S1, unofficial].

**When to Use:**
*   Use for novel view synthesis of reflective objects, geometry, materials, and lighting reconstruction [S1, unofficial].
*   Use when real-time relighting on mobile devices is required [S2, unofficial].

**When Not to Use:**
*   Not suitable for non-reflective or matte surfaces where PBR is not necessary, as it adds complexity [S1, unofficial].
*   Not suitable if coordinate system mismatches are not handled, leading to visual artifacts [S2, unofficial].

## Background

**Origin:**
*   Ref-Gaussian was introduced in the paper "Reflective Gaussian Splatting" [S1, unofficial].
*   It builds on 2D Gaussian Splatting (2DGS) and 3DGS-DR [S1, unofficial].

**History and Development:**
*   The method uses a two-stage training process: initial per-Gaussian rendering followed by deferred rendering training [S1, unofficial].
*   It employs material-aware normal propagation and edge-aware normal smoothness loss to improve geometry and normal accuracy [S1, unofficial].

**Usage:**
*   Used on Shiny Blender, Glossy Synthetic, and Ref-Real datasets [S1, unofficial].
*   Implemented for real-time rendering on iPhone using Metal [S2, unofficial].

## Sources

[S1] Reflective Gaussian Splatting, https://doi.org/10.48550/arxiv.2412.19282, fetched 2026-10-26 (unofficial)
[S2] iPhoneで Gaussian Splatting モデルの反射をリアルタイムに更新する, https://qiita.com/john-rocky/items/3c452476e66bdc29c290, fetched 2026-10-29 (unofficial)
