# Triplanar Texturing

Also called: ApplyTriplanarTexturing, TriPlanar, TriPlanarBump

## Overview
Triplanar texturing projects 2D textures onto a 3D surface from three orthogonal axes (X, Y, Z) and blends the results based on the surface normal, thereby avoiding the UV mapping artifacts and seams inherent in traditional parameterization [S1, S2, unofficial]. It is distinct from standard UV mapping by eliminating the need for explicit texture coordinates, making it ideal for procedural terrain and complex organic meshes where UV unwrapping is difficult or prone to distortion [S1, S2, unofficial].

## How it works
**Inputs:**
- World-space position ($P_{WS}$) and world-space normal ($N_{WS}$) of the surface point.
- Texture arrays or sets for Albedo, Normal, Metallic, Occlusion, Height, and Smoothness (MOHS) for each axis.
- Parameters: `_CoordinateScaling`, `_Tiling`, `_BlendOffset`, `_BlendHeightStrength`, `_BlendExponent`, `_Shininess`.

**Outputs:**
- Final Albedo color.
- Final Normal vector.
- Final Metallic, Occlusion, Height, and Smoothness values.

**Data Structures:**
The implementation uses two primary structures to manage intermediate data:

1.  `TriplanarUVs`: Contains three `float2` fields (`x`, `y`, `z`) representing the 2D texture coordinates for each projection axis.
2.  `TriplanarData`: Contains `half4 albedo`, `half3 normalWS`, `half metallic`, `half occlusion`, `half height`, and `half smoothness`. This structure aggregates the sampled properties before final blending.

**Step-by-Step Method:**

1.  **Calculate Triplanar UVs:**
    Compute the 2D coordinates for each axis based on the world-space position. The position components are scaled by `_CoordinateScaling` and multiplied by `_Tiling`. The `frac()` function is applied to ensure periodicity.
    -   X-axis UVs ($UV_x$): Derived from $P_{WS}.yz$.
    -   Y-axis UVs ($UV_y$): Derived from $P_{WS}.xz$.
    -   Z-axis UVs ($UV_z$): Derived from $P_{WS}.xy$.
    *Note: Signs may be adjusted for mirrored mapping, and offsets added to prevent repetition [S1, unofficial].*

2.  **Calculate Projection Weights:**
    Determine the influence of each axis based on the surface normal.
    -   Compute `projNormal`:
        $$ \text{projNormal} = \text{saturate}(\text{pow}(N_{WS} \times 1.4, 4)) $$
        where $\text{saturate}(x) = \max(0, \min(1, x))$ [S2, unofficial].
    -   Alternatively, weights can be derived from absolute normal components, adjusted by `_BlendOffset`, blended with height information using `_BlendHeightStrength`, raised to `_BlendExponent`, and then normalized [S1, unofficial].

3.  **Sample Textures:**
    Sample the texture maps for each axis using the calculated UVs.
    -   **Albedo:** Sample Albedo textures for X, Y, and Z axes.
    -   **Normals:** Sample Normal textures for X, Y, and Z axes. Unpack normal vectors from their storage format (e.g., RGB to [-1, 1]).
    -   **MOHS:** Sample Metallic, Occlusion, Height, and Smoothness textures for all three axes [S1, unofficial].
    -   **Y-Axis Selection:** For the Y-axis, select the "Top" texture if $N_{WS}.y > 0$, otherwise select the "Bottom" texture [S2, unofficial].

4.  **Blend Albedo:**
    Combine the albedo samples using the projection weights.
    -   Start with the Z-axis albedo ($A_z$).
    -   Lerp with X-axis albedo ($A_x$) using weight `projNormal.x`:
        $$ A_{xz} = \text{lerp}(A_z, A_x, \text{projNormal}.x) $$
    -   Lerp with Y-axis albedo ($A_y$) using weight `projNormal.y`:
        $$ A_{final} = \text{lerp}(A_{xz}, A_y, \text{projNormal}.y) $$
    *Note: Some implementations blend all three using normalized weights directly [S1, unofficial].*

5.  **Blend Normals (TriplanarBump):**
    Combine normal vectors using the projection weights.
    -   Let $N_x, N_y, N_z$ be the unpacked normal vectors.
    -   Lerp Z and X:
        $$ N_{zx} = \text{lerp}(N_z, N_x, \text{projNormal}.z) $$
    -   Lerp result with Y:
        $$ N_{final} = \text{lerp}(N_{zx}, N_y, \text{projNormal}.x) $$
    *Note: The specific weight assignment for normal blending may vary (e.g., using `projNormal.x` for the final lerp) [S2, unofficial]. Ensure normals are normalized after blending.*

6.  **Blend MOHS Properties:**
    Blend Metallic, Occlusion, Height, and Smoothness using the triplanar weights.
    -   Calculate weights $W_x, W_y, W_z$ based on $N_{WS}$ and optional height data [S1, unofficial].
    -   Normalize weights:
        $$ W_{sum} = W_x + W_y + W_z $$
        $$ W_x' = W_x / W_{sum}, \quad W_y' = W_y / W_{sum}, \quad W_z' = W_z / W_{sum} $$
    -   Blend each property $P$ (Metallic, Occlusion, Height, Smoothness):
        $$ P_{final} = P_x \times W_x' + P_y \times W_y' + P_z \times W_z' $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_triplanar_texturing(position_ws, normal_ws, textures, params):
    """
    position_ws: float3
    normal_ws: float3 (normalized)
    textures: dict containing 'albedo', 'normal', 'metallic', 'occlusion', 'height', 'smoothness'
              each key maps to a list/array of 3 textures [X, Y, Y_top/bottom]
    params: dict with keys 'CoordinateScaling', 'Tiling', 'BlendOffset', 'BlendHeightStrength', 'BlendExponent'
    """
    
    # 1. Calculate UVs
    # X-axis uses YZ, Y-axis uses XZ, Z-axis uses XY
    uv_x = frac((position_ws.yz * params['CoordinateScaling']) * params['Tiling'])
    uv_y = frac((position_ws.xz * params['CoordinateScaling']) * params['Tiling'])
    uv_z = frac((position_ws.xy * params['CoordinateScaling']) * params['Tiling'])
    
    # 2. Calculate Weights
    # Method from [S2, unofficial]: saturate(pow(normal * 1.4, 4))
    proj_normal = saturate(pow(normal_ws * 1.4, 4))
    
    # Method from [S1, unofficial] (alternative):
    # abs_normal = abs(normal_ws)
    # weights = abs_normal - params['BlendOffset']
    # if params['BlendHeightStrength'] > 0:
    #     # Blend with height if available
    #     pass 
    # weights = pow(weights, params['BlendExponent'])
    # weights = weights / sum(weights)
    
    # Using proj_normal weights for simplicity as per [S2, unofficial] dominant flow
    wx = proj_normal.x
    wy = proj_normal.y
    wz = proj_normal.z
    
    # 3. Sample Textures
    # Albedo
    albedo_x = sample_texture(textures['albedo'][0], uv_yz) # X-axis texture sampled with YZ coords
    albedo_y = sample_texture(textures['albedo'][1], uv_xz) # Y-axis texture sampled with XZ coords
    albedo_z = sample_texture(textures['albedo'][2], uv_xy) # Z-axis texture sampled with XY coords
    
    # Normal
    normal_x = unpack_normal(sample_texture(textures['normal'][0], uv_yz))
    normal_y = unpack_normal(sample_texture(textures['normal'][1], uv_xz))
    normal_z = unpack_normal(sample_texture(textures['normal'][2], uv_xy))
    
    # MOHS
    metallic_x = sample_texture(textures['metallic'][0], uv_yz)
    metallic_y = sample_texture(textures['metallic'][1], uv_xz)
    metallic_z = sample_texture(textures['metallic'][2], uv_xy)
    
    occlusion_x = sample_texture(textures['occlusion'][0], uv_yz)
    occlusion_y = sample_texture(textures['occlusion'][1], uv_xz)
    occlusion_z = sample_texture(textures['occlusion'][2], uv_xy)
    
    height_x = sample_texture(textures['height'][0], uv_yz)
    height_y = sample_texture(textures['height'][1], uv_xz)
    height_z = sample_texture(textures['height'][2], uv_xy)
    
    smoothness_x = sample_texture(textures['smoothness'][0], uv_yz)
    smoothness_y = sample_texture(textures['smoothness'][1], uv_xz)
    smoothness_z = sample_texture(textures['smoothness'][2], uv_xy)
    
    # 4. Blend Albedo
    # Lerp Z with X using projNormal.x, then with Y using projNormal.y
    albedo_xz = lerp(albedo_z, albedo_x, wx)
    albedo_final = lerp(albedo_xz, albedo_y, wy)
    
    # 5. Blend Normals
    # Lerp Z with X using projNormal.z, then with Y using projNormal.x (per [S2, unofficial])
    normal_zx = lerp(normal_z, normal_x, wz)
    normal_final = lerp(normal_zx, normal_y, wx)
    normal_final = normalize(normal_final)
    
    # 6. Blend MOHS
    # Simple weighted average using proj_normal components
    metallic_final = metallic_x * wx + metallic_y * wy + metallic_z * wz
    occlusion_final = occlusion_x * wx + occlusion_y * wy + occlusion_z * wz
    height_final = height_x * wx + height_y * wy + height_z * wz
    smoothness_final = smoothness_x * wx + smoothness_y * wy + smoothness_z * wz
    
    return {
        'albedo': albedo_final,
        'normal': normal_final,
        'metallic': metallic_final,
        'occlusion': occlusion_final,
        'height': height_final,
        'smoothness': smoothness_final
    }

def sample_texture(tex, uv):
    # Placeholder for texture sampling logic
    pass

def unpack_normal(n):
    # Placeholder for normal unpacking (e.g., RGB to [-1,1])
    pass

def saturate(x):
    return max(0.0, min(1.0, x))

def lerp(a, b, t):
    return a + t * (b - a)
```

**Edge Cases and Pitfalls:**
-   **Zero Weights:** If the normal is perfectly aligned with an axis, other weights may become zero. Ensure normalization of weights if using the absolute-normal method to avoid division by zero or zero-weight artifacts [S1, unofficial].
-   **Normal Unpacking:** Incorrect unpacking of normal maps (e.g., assuming linear vs. sRGB) will cause visual artifacts [S1, unofficial].
-   **Mirrored Mapping:** For seamless tiling, mirrored mapping may be required to avoid seams at texture boundaries. Signs of UVs must be adjusted accordingly [S1, unofficial].
-   **Y-Axis Direction:** The Y-axis texture selection (Top vs. Bottom) depends on the normal's Y component. Failing to handle this can cause upside-down textures on ceilings or floors [S2, unofficial].

## Operating characteristics
-   **Time Complexity:** $O(1)$ per pixel, as it involves a fixed number of texture samples (typically 9 for full MOHS + Normal + Albedo) and arithmetic operations.
-   **Memory Complexity:** $O(1)$ additional memory per pixel for intermediate UVs and weights. Texture memory usage depends on the number of texture sets (3x per property).
-   **Performance:** Generally fast due to lack of UV calculation overhead and efficient branching. However, sampling 9 textures per pixel can be memory-bandwidth intensive [S1, S2, unofficial].
-   **Strengths:** No UV seams, works on any geometry, easy to tile procedurally.
-   **Limitations:** Can cause blurring or stretching on surfaces with high curvature if not blended carefully. Does not account for perspective distortion in the same way as UV mapping.
-   **When to Use:** Procedural terrain, organic models, or any scenario where UV unwrapping is impractical [S1, S2, unofficial].
-   **When Not to Use:** When precise texture alignment is required (e.g., architectural details) or when memory bandwidth is a critical bottleneck.

## Background
Triplanar texturing is a technique widely used in procedural generation and real-time rendering to apply textures to 3D surfaces without explicit UV coordinates. It is particularly prevalent in game development for terrain rendering. The method is implemented in various engines and libraries, including Unity-based projects like Tuntenfisch/Voxels and theSoenke/ProceduralTerrain [S1, S2, unofficial].

## Sources
[S1] Tantenfisch/Voxels, https://github.com/Tantenfisch/Voxels, fetched 2026-10-26 (unofficial)
[S2] theSoenke/ProceduralTerrain, https://github.com/theSoenke/ProceduralTerrain, fetched 2026-10-27 (unofficial)
