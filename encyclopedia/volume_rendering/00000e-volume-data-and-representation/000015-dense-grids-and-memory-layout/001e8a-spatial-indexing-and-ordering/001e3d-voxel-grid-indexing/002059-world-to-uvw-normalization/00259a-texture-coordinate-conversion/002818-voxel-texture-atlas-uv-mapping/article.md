# Voxel Texture Atlas UV Mapping


## Overview
Voxel Texture Atlas UV Mapping is a technique that maps voxel face types and directions to a 2D texture atlas using calculated UV coordinates, incorporating safety margins to prevent texture bleeding [S2, unofficial]. It builds on [Texture Coordinate Conversion](3581-texture-coordinate-conversion.md) by extending the coordinate conversion into a structured atlas layout, allowing efficient sampling of multiple voxel types from a single texture resource [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Voxel type ID ($v\_Type$), face direction index ($v\_Direction \in [0,5]$), vertex UV coordinate within the quad ($v\_TexCoord \in [0,1]$), and the texture atlas image.
*   **Outputs:** Final UV coordinates ($finalUV$) and sampled color with applied effects (transparency, shadow).

**Data Structures:**
*   **Texture Atlas:** A 2D image of size $512 \times 512$ pixels [S2, unofficial].
*   **Tile Layout:**
    *   Tile Size: $16 \times 16$ pixels [S2, unofficial].
    *   Border Size: $1$ pixel [S2, unofficial].
    *   Total Tile Size: $17 \times 17$ pixels (including border) [S2, unofficial].
    *   Tiles per Row/Column: 30 [S2, unofficial].
*   **Face Data Packing:** Voxel face data is packed into a 32-bit integer (or float) with the following bit layout [S2, unofficial]:
    *   Bits 0-3: Direction (3 bits, value 0-5)
    *   Bits 4-11: Type (8 bits, value 0-255)
    *   Bits 12-15: Z coordinate (4 bits, value 0-15)
    *   Bits 16-19: Y coordinate (4 bits, value 0-15)
    *   Bits 20-23: X coordinate (4 bits, value 0-15)
    *   Packing formula: $packed = (x \ll 20) | (y \ll 16) | (z \ll 12) | (type \ll 4) | (direction \ \& \ 0b111)$ [S2, unofficial].

**Steps:**
1.  **Cache Check:** Check for a cached UV file at `cache_path + '_uv.npz'`. If it exists, load vertex texture ($vt$), face texture ($ft$), and vertex mapping ($vmapping$) arrays directly [S1, unofficial].
2.  **Base UV Calculation:** Calculate the base UV for the tile corresponding to the voxel type/direction.
    *   Determine row and column indices from the face index or type.
    *   $baseUV.x = (col \times TOTAL\_TILE\_SIZE + BORDER\_SIZE) / ATLAS\_WIDTH$
    *   $baseUV.y = 1.0 - ((row \times TOTAL\_TILE\_SIZE + TOTAL\_TILE\_SIZE - BORDER\_SIZE) / ATLAS\_HEIGHT)$ [S2, unofficial].
3.  **Safe UV Range Calculation:** Calculate the safe area within the tile to prevent bleeding from adjacent tiles.
    *   $safeTileSize.x = (TILE\_SIZE - 2.0 \times BORDER\_SIZE) / ATLAS\_WIDTH$
    *   $safeTileSize.y = (TILE\_SIZE - 2.0 \times BORDER\_SIZE) / ATLAS\_HEIGHT$
    *   $SAFE\_MARGIN = 0.002$ (in UV space) [S2, unofficial].
4.  **Within-Tile UV Calculation:** Map the vertex UV coordinate to the safe area.
    *   $withinTileUV.x = v\_TexCoord.x \times safeTileSize.x + SAFE\_MARGIN$
    *   $withinTileUV.y = v\_TexCoord.y \times safeTileSize.y + SAFE\_MARGIN$ [S2, unofficial].
5.  **Final UV Assembly:**
    *   $finalUV = tileBaseUV + withinTileUV$ [S2, unofficial].
6.  **Vertex Remapping (Optional):** If vertex mapping ($vmap$) is enabled, remap vertex and normal arrays to align with UV indices using `align_v_to_vt(vmapping)`, ensuring each vertex has a unique UV coordinate [S1, unofficial].
7.  **Texture Sampling:** Sample the texture atlas using the final UV.
    *   $color = texture2D(textureAtlas, finalUV)$ [S2, unofficial].
8.  **Post-Processing:**
    *   **Water Transparency:** If $v\_Type == 5.0$, set $color.a = 0.5$ [S2, unofficial].
    *   **Directional Shadow:** Calculate shadow factor based on face direction.
        *   $shadowFactor = \max(0.5, \dot(normalize(vec3(v\_Direction, 0.5, 1.0)), vec3(0.0, 1.0, 1.0)))$
        *   $color.rgb *= shadowFactor$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_voxel_uv(v_type, v_direction, v_tex_coord, atlas_width=512, atlas_height=512, tile_size=16, border_size=1, safe_margin=0.002):
    """
    Calculate final UV coordinates for a voxel face in a texture atlas.
    
    Args:
        v_type (int): Voxel type ID.
        v_direction (int): Face direction index (0-5).
        v_tex_coord (vec2): Vertex UV coordinate within the quad (0-1).
        
    Returns:
        vec2: Final UV coordinates.
    """
    # Determine tile position based on type and direction
    # Note: The exact mapping of type/direction to row/col is implementation-specific.
    # Assuming a simple linear mapping for demonstration, or lookup from a precomputed map.
    # For this example, we assume face_index determines the tile.
    face_index = v_type * 6 + v_direction # Hypothetical mapping
    total_tiles_per_row = atlas_width // (tile_size + 2 * border_size)
    
    col = face_index % total_tiles_per_row
    row = face_index // total_tiles_per_row
    
    total_tile_size = tile_size + 2 * border_size
    
    # Base UV calculation
    base_uv_x = (col * total_tile_size + border_size) / atlas_width
    base_uv_y = 1.0 - ((row * total_tile_size + total_tile_size - border_size) / atlas_height)
    
    # Safe UV range calculation
    safe_tile_size_x = (tile_size - 2.0 * border_size) / atlas_width
    safe_tile_size_y = (tile_size - 2.0 * border_size) / atlas_height
    
    # Within-tile UV calculation
    within_tile_uv_x = v_tex_coord.x * safe_tile_size_x + safe_margin
    within_tile_uv_y = v_tex_coord.y * safe_tile_size_y + safe_margin
    
    # Final UV
    final_uv_x = base_uv_x + within_tile_uv_x
    final_uv_y = base_uv_y + within_tile_uv_y
    
    return vec2(final_uv_x, final_uv_y)

def apply_voxel_effects(color, v_type, v_direction):
    """
    Apply water transparency and directional shadow to the sampled color.
    
    Args:
        color (vec4): Sampled color from texture atlas.
        v_type (int): Voxel type ID.
        v_direction (int): Face direction index (0-5).
        
    Returns:
        vec4: Modified color.
    """
    # Water transparency
    if v_type == 5.0:
        color.a = 0.5
        
    # Directional shadow
    # Note: v_direction is an integer 0-5. The source uses it as a component in a vec3.
    # This implies a specific mapping of direction index to a 3D vector.
    # The source formula: dot(normalize(vec3(v_Direction, 0.5, 1.0)), vec3(0.0, 1.0, 1.0))
    # This is likely a simplified or specific implementation detail.
    # We will implement it exactly as stated.
    
    # Create a vector from direction. Since direction is 0-5, we need to map it to a 3D vector.
    # The source code uses `vec3(v_Direction, 0.5, 1.0)`. This suggests v_Direction is treated as a scalar component.
    # However, direction is an index. Let's assume the source meant a specific vector for each direction.
    # Given the ambiguity, we follow the literal formula provided in the source notes.
    
    # Re-reading source: "shadowFactor = max(0.5, dot(normalize(vec3(v_Direction, 0.5, 1.0)), vec3(0.0, 1.0, 1.0)))"
    # This formula uses v_Direction directly as a float component. This is unusual for an index 0-5.
    # It might be a typo in the source notes or a specific encoding.
    # We will implement it as written, assuming v_Direction is cast to float.
    
    dir_vec = vec3(float(v_direction), 0.5, 1.0)
    light_vec = vec3(0.0, 1.0, 1.0)
    
    shadow_factor = max(0.5, dot(normalize(dir_vec), normalize(light_vec)))
    
    color.rgb *= shadow_factor
    
    return color
```

**Edge Cases and Pitfalls:**
*   **Texture Bleeding:** Without the safety margin ($SAFE\_MARGIN$), sampling near the edges of a tile may pick up pixels from adjacent tiles due to filtering or precision errors [S2, unofficial].
*   **Precision:** The bit-packed face data must be unpacked correctly to retrieve the voxel type and direction. Incorrect bit shifts or masks will lead to wrong tile selection [S2, unofficial].
*   **Boundary Conditions:** The UV coordinates must be clamped to $[0,1]$ to avoid out-of-bounds sampling, although the atlas layout and safe margins should theoretically prevent this [S2, unofficial].

## Subtopics

### [Voxel-based Surface Parameterization](10557-voxel-based-surface-parameterization.md)
A method to transform the world coordinate system to a system defined by one voxel, ensuring every component of every point $v_{hit}$ is in $[0,1]$. It determines texture coordinates based on the surface normal and the voxel side-plane whose normal is most similar to the surface normal. Voxels are never created explicitly [10557-voxel-based-surface-parameterization.md].

### Voxel UV Unwrapping for AO Interpolation
A variant that calculates texture coordinates using modulo arithmetic on the hit position and normal to interpolate Ambient Occlusion (AO) values from four face AO values. It uses bilinear interpolation with the calculated UV and applies gamma correction to the AO value [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per voxel face for UV calculation, assuming constant-time lookup for tile position.
*   **Memory Complexity:** $O(1)$ for UV calculation, plus the memory for the texture atlas ($512 \times 512$ pixels) [S2, unofficial].
*   **Performance:** Efficient for rendering large voxel worlds as it reduces texture bind calls and allows batching of voxel faces. The bit-packing allows compact storage of face data [S2, unofficial].
*   **Strengths:** Prevents texture bleeding, supports multiple voxel types in a single texture, allows for efficient caching [S1, unofficial][S2, unofficial].
*   **Limitations:** Requires pre-computation of the atlas layout and UV mappings. The fixed tile size and atlas dimensions limit the number of voxel types and directions that can be supported without changing the atlas [S2, unofficial].

## Background
The technique is used in web-based voxel terrain rendering and 3D content generation frameworks. It is implemented in projects like `web_voxel_terrain` for efficient rendering of voxel chunks and `dreamgaussian` for 3D content generation [S1, unofficial][S2, unofficial]. The bit-packing strategy is also used in `bevy-voxel-engine` for compact storage of voxel data [S3, unofficial].

## Sources
[S1] dreamgaussian/dreamgaussian, https://github.com/dreamgaussian/dreamgaussian, fetched 2026-10-27 (unofficial)
[S2] andrewbloese-00/web_voxel_terrain, https://github.com/andrewbloese-00/web_voxel_terrain, fetched 2026-10-27 (unofficial)
[S3] ria8651/bevy-voxel-engine, https://github.com/ria8651/bevy-voxel-engine, fetched 2026-10-27 (unofficial)
