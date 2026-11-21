# Voxel Texture Atlas UV Mapping


## Overview
Voxel Texture Atlas UV Mapping is a technique for mapping voxel face types and directions to a 2D texture atlas using calculated UV coordinates, with safety margins to prevent texture bleeding [S3, unofficial]. It differs from standard UV unwapping by deterministically selecting the visible face based on geometric normals and handling the discrete block structure explicitly, ensuring consistent texture alignment across adjacent voxels [S3, unofficial]. It is primarily used to sample textures from a single atlas for meshed voxel geometry, often in conjunction with greedy meshing and visible-face culling [S1, S3, unofficial].

## How it works
**Inputs:**
- A voxel grid or chunk containing voxel data.
- A 2D texture atlas image (e.g., 512x512 pixels).
- Configuration parameters: `ATLAS_WIDTH`, `ATLAS_HEIGHT`, `TILE_SIZE`, `BORDER_SIZE`, `SAFE_MARGIN`.

**Outputs:**
- A mesh with vertex attributes: position, normal, `vt` (vertex texture coordinates), `ft` (face/vertex mapping indices), and `vmapping` (vertex-to-UV mapping).
- A shader material that samples the texture atlas using the computed UVs.

**Data Structures:**
1. **Packed Face Data:** A 32-bit integer encoding voxel properties to minimize memory and bandwidth.
   - Bits 0-2: `direction` (3 bits, face index 0-5).
   - Bits 3-10: `type` (8 bits, voxel type ID 0-255).
   - Bits 11-14: `z` (4 bits, local Z coordinate 0-15).
   - Bits 15-18: `y` (4 bits, local Y coordinate 0-15).
   - Bits 19-22: `x` (4 bits, local X coordinate 0-15).
   - Layout: `(x << 20) | (y << 16) | (z << 12) | (type << 4) | (direction & 0b111)` [S3, unofficial].
2. **Texture Atlas Configuration:**
   - `ATLAS_WIDTH`, `ATLAS_HEIGHT`: Total dimensions (e.g., 512).
   - `TILE_SIZE`: Size of one texture tile (e.g., 16).
   - `BORDER_SIZE`: Size of the safety border around each tile (e.g., 1).
   - `TOTAL_TILE_SIZE`: `TILE_SIZE + 2 * BORDER_SIZE` (e.g., 17).
   - `SAFE_MARGIN`: Offset in UV space to avoid bleeding (e.g., 0.002) [S3, unofficial].

**Steps:**

1. **Cache Check:** Check for a cached UV file at `cache_path + '_uv.npz'`. If it exists, load `vt`, `ft`, and `vmapping` arrays directly [S2, unofficial].

2. **Mesh Generation & Face Culling:**
   - Iterate through the voxel grid/chunk.
   - Identify visible faces using greedy meshing or visible-face culling [S1, S3, unofficial].
   - For each visible face, determine the `direction` (0-5) and `type` (voxel ID) [S3, unofficial].
   - Pack the face data into a 32-bit integer using the layout defined above [S3, unofficial].

3. **UV Coordinate Calculation:**
   - Determine the tile position in the atlas based on the `type` or `direction`. Let `col` and `row` be the tile coordinates in the atlas grid.
   - Calculate the base UV for the tile:
     $$ baseUV.x = \frac{col \times TOTAL\_TILE\_SIZE + BORDER\_SIZE}{ATLAS\_WIDTH} $$
     $$ baseUV.y = 1.0 - \frac{row \times TOTAL\_TILE\_SIZE + TOTAL\_TILE\_SIZE - BORDER\_SIZE}{ATLAS\_HEIGHT} $$
     Note: The Y-axis is flipped to match standard texture coordinate systems where (0,0) is bottom-left [S3, unofficial].
   - Calculate the safe tile size in UV space:
     $$ safeTileSize.x = \frac{TILE\_SIZE - 2.0 \times BORDER\_SIZE}{ATLAS\_WIDTH} $$
     $$ safeTileSize.y = \frac{TILE\_SIZE - 2.0 \times BORDER\_SIZE}{ATLAS\_HEIGHT} $$
   - For each vertex of the face quad, calculate the local UV coordinate `v_TexCoord` (0-1) within the quad.
   - Apply the safe margin to prevent texture bleeding:
     $$ withinTileUV.x = v\_TexCoord.x \times safeTileSize.x + SAFE\_MARGIN $$
     $$ withinTileUV.y = v\_TexCoord.y \times safeTileSize.y + SAFE\_MARGIN $$
   - Compute the final UV:
     $$ finalUV = baseUV + withinTileUV $$
   - Store `finalUV` in the `vt` attribute [S3, unofficial].

4. **Vertex Remapping (Optional):**
   - If `vmap` is True, remap vertex and normal arrays to align with UV indices using `align_v_to_vt(vmapping)`. This ensures each vertex has a unique UV coordinate, which is critical for correct texture sampling and normal interpolation [S2, unofficial].

5. **Shader Sampling:**
   - In the fragment shader, sample the texture atlas:
     $$ color = texture2D(textureAtlas, finalUV) $$
   - Apply additional effects such as water transparency (if `v_Type == 5.0`, `color.a = 0.5`) and directional shadowing:
     $$ shadowFactor = \max(0.5, \dot(\text{normalize}(\vec{3}(v\_Direction, 0.5, 1.0)), \vec{3}(0.0, 1.0, 1.0))) $$
     $$ color.rgb *= shadowFactor $$
     [S3, unofficial].

**Parameters:**
- `v_Type`: Varying float, voxel type ID [S3, unofficial].
- `v_Direction`: Varying float, face direction index (0-5) [S3, unofficial].
- `v_TexCoord`: Varying vec2, vertex UV coordinate within the quad (0-1) [S3, unofficial].

**Edge Cases and Pitfalls:**
- **Texture Bleeding:** Without `BORDER_SIZE` and `SAFE_MARGIN`, adjacent tiles may bleed into each other due to filtering or precision errors.
- **Vertex Uniqueness:** If multiple faces share a vertex but have different UVs, the vertex must be duplicated or remapped (`vmap`) to avoid incorrect texture sampling [S2, unofficial].
- **Cache Consistency:** If the voxel grid changes, the cache must be invalidated and regenerated [S2, unofficial].

## Subtopics

## Operating characteristics
- **Time Complexity:** Depends on the meshing algorithm (greedy meshing is generally $O(N)$ where $N$ is the number of voxels). UV calculation is $O(F)$ where $F$ is the number of faces.
- **Memory Complexity:** $O(V)$ for voxel data, $O(F)$ for mesh data. The packed face data reduces memory footprint [S3, unofficial].
- **Performance:** Caching UV data significantly speeds up repeated renders [S2, unofficial]. Using a single texture atlas reduces texture bind calls [S1, unofficial].
- **Strengths:** Efficient memory usage, consistent texture alignment, easy to implement with standard WebGL/OpenGL.
- **Limitations:** Limited by atlas size; large numbers of voxel types require larger atlases or texture arrays.

## Background
This method is commonly used in web-based voxel engines and games, leveraging greedy meshing and visible-face culling to optimize rendering [S1, S3, unofficial]. It is often implemented in JavaScript/TypeScript for browser-based applications [S1, S3, unofficial].

## Sources
[S1] Phase 2: voxel engine com chunks, meshing e edição, https://github.com/EduardoLavall/blockbound/issues/4, fetched 2026-10-26 (unofficial)
[S2] dreamgaussian/dreamgaussian, https://github.com/dreamgaussian/dreamgaussian, fetched 2026-10-27 (unofficial)
[S3] andrewbloese-00/web_voxel_terrain, https://github.com/andrewbloese-00/web_voxel_terrain, fetched 2026-10-27 (unofficial)
