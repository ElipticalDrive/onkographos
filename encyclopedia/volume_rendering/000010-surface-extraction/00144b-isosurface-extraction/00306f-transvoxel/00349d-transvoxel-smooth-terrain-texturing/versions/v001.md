# Transvoxel Smooth Terrain Texturing

Also called: Transvoxel Terrain Texturing, Smooth Material Texturing

## Overview
This method applies triplanar texturing to surfaces extracted by the Transvoxel algorithm, specifically designed for terrain rendering where material properties vary based on surface orientation. It blends between "top" and "side" textures (or a material array) using a "topness" factor derived from the surface normal, ensuring seamless texture projection regardless of the underlying voxel resolution or LOD transitions. It builds on [Transvoxel](6277-transvoxel.md) for geometry generation.

## How it works
**Inputs:**
- Vertex position in local space (`VERTEX`).
- Vertex normal in local space (`NORMAL`).
- Model matrix (`MODEL_MATRIX`) and View matrix (`VIEW_MATRIX`).
- Texture samplers: `u_texture_top` (top material), `u_texture_sides` (side material), or `u_albedo_array` (material array).
- Parameters: `triplanar_sharpness` (default 8.0), `topness_midlevel` (0.5), `topness_smoothness` (0.04).

**Outputs:**
- Fragment color (`ALBEDO`).
- Fragment alpha (`ALPHA`, optional).

**Data Structures:**
- Varying variables passed from vertex to fragment shader:
  - `v_world_pos`: World-space position of the vertex.
  - `v_vertex_pos_model`: Model-space vertex position (used for normal computation).
  - `v_material_indices`: (Optional) Indices for material array blending.
  - `v_material_weights`: (Optional) Weights for material array blending.

**Steps:**

1. **Position Computation:**
   Compute the final vertex position using the Transvoxel helper function:
   ```python
   VERTEX = get_transvoxel_position(VERTEX, CUSTOM0)
   ```
   Compute the world-space position:
   ```python
   v_world_pos = (MODEL_MATRIX * vec4(VERTEX, 1.0)).xyz
   ```

2. **Normal Computation:**
   Compute the surface normal in view space using the Transvoxel helper, which accounts for LOD transitions:
   ```python
   NORMAL = get_voxel_normal_view(v_vertex_pos_model, NORMAL, VIEW_MATRIX * MODEL_MATRIX)
   ```
   Transform the normal to world space for triplanar mapping:
   ```python
   normal = (INV_VIEW_MATRIX * vec4(NORMAL, 0.0)).xyz
   ```

3. **Topness Calculation:**
   Calculate a "topness" factor based on the Y-component of the world-space normal. This determines how much of the "top" texture is visible versus the "side" texture.
   ```python
   topness_min = topness_midlevel - topness_smoothness
   topness_max = topness_midlevel + topness_smoothness
   topness = smoothstep(topness_min, topness_max, normal.y)
   ```
   Where `topness_midlevel = 0.5` and `topness_smoothness = 0.04`.

4. **Triplanar Blending Factors:**
   Compute blending weights for the X, Y, and Z axes based on the normal and sharpness parameter:
   ```python
   blending = get_triplanar_blend(normal, triplanar_sharpness)
   ```
   Where `triplanar_sharpness = 8.0`.

5. **Texture Sampling:**
   Scale the world position for texture sampling:
   ```python
   wpos = v_world_pos * 0.2
   ```
   Sample the top and side textures using triplanar mapping:
   ```python
   top_col = texture_triplanar(u_texture_top, wpos, blending).rgb
   side_col = texture_triplanar(u_texture_sides, wpos, blending).rgb
   ```

6. **Color Blending:**
   Mix the top and side colors based on the topness factor:
   ```python
   ALBEDO = mix(side_col, top_col, topness)
   ```

   **Alternative: Material Array Blending:**
   If using a material array (`u_albedo_array`), sample the array using material indices and weights:
   ```python
   col0 = texture_triplanar(u_albedo_array, wpos, blending, v_material_indices.x).rgb
   col1 = texture_triplanar(u_albedo_array, wpos, blending, v_material_indices.y).rgb
   col2 = texture_triplanar(u_albedo_array, wpos, blending, v_material_indices.z).rgb
   col3 = texture_triplanar(u_albedo_array, wpos, blending, v_material_indices.w).rgb

   weights = v_material_weights
   weights /= (weights.x + weights.y + weights.z + weights.w + 0.00001)

   ALBEDO = col0 * weights.r + col1 * weights.g + col2 * weights.b + col3 * weights.a
   ```

7. **LOD Fade Discard:**
   Discard fragments that are in the LOD transition zone to avoid visual artifacts:
   ```python
   if (get_lod_fade_discard(SCREEN_UV)) {
       discard
   }
   ```

**Edge Cases and Pitfalls:**
- **Precision:** The topness calculation uses `smoothstep` with a narrow range (`0.46` to `0.54`). Ensure normal vectors are normalized before use.
- **Boundaries:** The LOD fade discard check prevents rendering in regions where the geometry is being transitioned between LOD levels.
- **Degenerate Input:** If the normal is zero or undefined, the topness and blending factors may be incorrect. The Transvoxel helper should handle this, but fallback values should be considered.

## Subtopics

## Operating characteristics
- **Time Complexity:** O(1) per vertex and fragment, dominated by texture sampling and matrix multiplications.
- **Memory Complexity:** O(1) additional memory per vertex/fragment, plus texture memory for top/side textures or material array.
- **Performance:** Efficient on modern GPUs due to parallel texture sampling. The use of triplanar mapping avoids UV unwrapping overhead.
- **Strengths:** Seamless texturing across LOD boundaries, orientation-aware material blending, simple implementation.
- **Limitations:** Triplanar mapping can cause stretching on highly curved surfaces; topness factor is a simple heuristic and may not capture complex material distributions.

## Background
This method is part of the Zylann's VoxelGame project, an open-source voxel engine for Godot. It extends the Transvoxel algorithm to provide high-quality terrain texturing. The approach is documented in the project's source code and shader files [S1, unofficial].

## Sources
[S1] Zylann/voxelgame, https://github.com/Zylann/voxelgame, fetched 2026-10-26 (unofficial)
