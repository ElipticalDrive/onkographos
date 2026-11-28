# Smooth Material Texturing with Texture Array

   Also called: Transvoxel Smooth Terrain Texturing with Texture Array

   ## Overview
   This method applies multi-material texturing to surfaces extracted by the Transvoxel algorithm using a texture array and triplanar mapping [S1, unofficial]. It supports up to four distinct materials per vertex, blended via weights stored in vertex attributes, and uses triplanar projection to avoid UV seams on complex geometry [S1, unofficial].

   ## How it works
   **Inputs:**
   - Vertex position (`VERTEX`), normal (`NORMAL`), and custom attributes (`CUSTOM0`, `CUSTOM1`) [S1, unofficial].
   - `CUSTOM0`: Transvoxel position data [S1, unofficial].
   - `CUSTOM1`: Material data packed in `vec4`. `x` stores 4 integer material indices (8-bit each); `y` stores 4 material weights (8-bit each) [S1, unofficial].
   - `u_albedo_array`: A `sampler2DArray` containing the albedo textures for the materials [S1, unofficial].
   - `MODEL_MATRIX`, `INV_VIEW_MATRIX`: Transformation matrices [S1, unofficial].

   **Outputs:**
   - `ALBEDO`: The final blended color [S1, unofficial].
   - `v_world_pos`: World-space position [S1, unofficial].
   - `v_material_indices`: Decoded material indices [S1, unofficial].
   - `v_material_weights`: Normalized material weights [S1, unofficial].

   **Data Structures:**
   - `CUSTOM1.x`: Packed 4x8-bit integers. Each byte represents an index into the texture array [S1, unofficial].
   - `CUSTOM1.y`: Packed 4x8-bit integers. Each byte represents a weight in the range [0, 255] [S1, unofficial].

   **Steps:**
   1. **Vertex Shader:**
      - Compute the final vertex position using the Transvoxel helper: `get_transvoxel_position(VERTEX, CUSTOM0)` [S1, unofficial].
      - Transform the vertex to world space: `v_world_pos = (MODEL_MATRIX * vec4(VERTEX, 1.0)).xyz` [S1, unofficial].
      - Decode the material indices from `CUSTOM1.x` into `v_material_indices` using `decode_8bit_vec4` [S1, unofficial].
      - Decode the material weights from `CUSTOM1.y` into `v_material_weights` using `decode_8bit_vec4`, then normalize to [0, 1] by dividing by 255.0 [S1, unofficial].

   2. **Fragment Shader:**
      - Transform the surface normal to world space: `normal = (INV_VIEW_MATRIX * vec4(NORMAL, 0.0)).xyz` [S1, unofficial].
      - Define a texture scale `uv_scale = 1.0` and compute the world position for sampling: `wpos = v_world_pos * uv_scale` [S1, unofficial].
      - Compute triplanar blending factors: `blending = get_triplanar_blend(normal, 1.0)` [S1, unofficial].
      - Sample the texture array for each of the 4 material indices using `texture_array_triplanar(u_albedo_array, wpos, blending, index)` to get `col0` through `col3` [S1, unofficial].
      - Normalize the material weights by dividing by the sum of all weights plus a small epsilon `0.00001` to prevent division by zero: `weights /= (weights.x + weights.y + weights.z + weights.w + 0.00001)` [S1, unofficial].
      - Calculate the final albedo by blending the 4 sampled colors with their normalized weights: `col = col0 * weights.r + col1 * weights.g + col2 * weights.b + col3 * weights.a` [S1, unofficial].

   **Pseudocode:**
   ```python
   # Vertex Shader
   def vertex(VERTEX, CUSTOM0, CUSTOM1, MODEL_MATRIX):
       VERTEX = get_transvoxel_position(VERTEX, CUSTOM0)
       v_world_pos = (MODEL_MATRIX * vec4(VERTEX, 1.0)).xyz
       v_material_indices = decode_8bit_vec4(CUSTOM1.x)
       v_material_weights = decode_8bit_vec4(CUSTOM1.y) / 255.0

   # Fragment Shader
   def fragment(v_world_pos, v_material_indices, v_material_weights, NORMAL, INV_VIEW_MATRIX, u_albedo_array):
       normal = (INV_VIEW_MATRIX * vec4(NORMAL, 0.0)).xyz
       uv_scale = 1.0
       wpos = v_world_pos * uv_scale
       blending = get_triplanar_blend(normal, 1.0)
       
       col0 = texture_array_triplanar(u_albedo_array, wpos, blending, v_material_indices.x).rgb
       col1 = texture_array_triplanar(u_albedo_array, wpos, blending, v_material_indices.y).rgb
       col2 = texture_array_triplanar(u_albedo_array, wpos, blending, v_material_indices.z).rgb
       col3 = texture_array_triplanar(u_albedo_array, wpos, blending, v_material_indices.w).rgb
       
       weight_sum = v_material_weights.x + v_material_weights.y + v_material_weights.z + v_material_weights.w + 0.00001
       normalized_weights = v_material_weights / weight_sum
       
       albedo = (col0 * normalized_weights.r + 
                 col1 * normalized_weights.g + 
                 col2 * normalized_weights.b + 
                 col3 * normalized_weights.a)
       return albedo

   # Helper functions
   def get_transvoxel_position(vert, custom):
       # Not given in the sources [G]
       return vert

   def decode_8bit_vec4(val):
       return vec4(floor(mod(val * 255.0, 256.0)))

   def get_triplanar_blend(normal, scale):
       abs_n = abs(normal)
       return normalize(abs_n)

   def texture_array_triplanar(array, wpos, blending, index):
       uv_xy = wpos.xy
       uv_xz = wpos.xz
       uv_yz = wpos.yz
       s_xy = texture(array, vec3(uv_xy, index)).rgb
       s_xz = texture(array, vec3(uv_xz, index)).rgb
       s_yz = texture(array, vec3(uv_yz, index)).rgb
       return s_xy * blending.x + s_xz * blending.y + s_yz * blending.z
   ```

   **Parameters:**
   - `uv_scale`: Default `1.0` [S1, unofficial].
   - `epsilon`: `0.00001` to prevent division by zero [S1, unofficial].

   **Edge Cases:**
   - **Zero Weights:** The epsilon ensures no division by zero occurs if all weights are zero [S1, unofficial].
   - **Boundary Conditions:** Triplanar mapping handles boundaries naturally, but texture array indexing must be valid (0 to N-1) [S1, unofficial].

   ## Operating characteristics
   - **Time Complexity:** O(1) per fragment, with 4 texture array samples [S1, unofficial].
   - **Memory Complexity:** O(N) for the texture array, where N is the number of materials [S1, unofficial].
   - **Strengths:** Seamless blending, no UV seams, supports dynamic material changes [S1, unofficial].
   - **Limitations:** Limited to 4 materials per vertex, texture array size constraints [S1, unofficial].

   ## Background
   Originated in the Zylann/voxelgame repository [S1, unofficial], this method extends the Transvoxel algorithm to support smooth material texturing. It is used in voxel-based games and simulations where dynamic terrain and material blending are required [S1, unofficial].

   ## Sources
   [S1, unofficial] Zylann/voxelgame, https://github.com/Zylann/voxelgame, fetched 2026-10-26 (unofficial)
