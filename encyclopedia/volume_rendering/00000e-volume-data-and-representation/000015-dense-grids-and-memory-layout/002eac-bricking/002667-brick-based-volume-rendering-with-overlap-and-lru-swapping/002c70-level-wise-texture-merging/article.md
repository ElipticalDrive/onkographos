# Level-wise Texture Merging

Also called: Octree-like hierarchy of texture maps, Texture hierarchy

## Overview
Level-wise texture merging minimizes texture memory overhead in hierarchical volume rendering by merging the texture data of adjacent bricks into single, larger texture maps [S1, S2]. It creates an octree-like hierarchy where the number of bricks sharing a single texture map is multiplied by a factor of 8 on each consecutive level [S1, S2]. This method reduces the number of distinct texture objects required for coarse levels of detail compared to assigning individual textures to every brick [S1, S2].

## How it works
**Inputs and Outputs**
*   **Input:** A set of 3D volume bricks organized into a hierarchical structure with $l$ levels. The number of bricks in each spatial direction ($N_x, N_y, N_z$) should ideally be a power of two [S1, S2].
*   **Output:** A texture hierarchy where higher-level textures contain merged data from lower-level bricks, managed by a texture manager [S1, S2].

**Data Structures**
*   **Texture Manager:** An object responsible for the administration of the texture hierarchy [S1, S2]. It creates the desired texture maps and assigns appropriate sub-textures to bricks requesting texture memory [S1, S2].
*   **Texture Maps:** 3D textures. The same physical texture size is used on every level of the hierarchy [S1, S2].
*   **Brick-to-Texture Mapping:** Each brick is assigned a sub-region (sub-texture) within a parent texture map based on its level and position [S1, S2].

**Algorithm Steps**
1.  **Initialize Level 0:** Assign a unique, dedicated texture map to each brick at the finest level (level 0) [S1, S2].
2.  **Merge for Level 1:** Group eight adjacent bricks from level 0 into a single parent group [S1, S2]. Create one texture map for this group. The eight bricks share this single texture map, using appropriate texture coordinates to address their respective sub-volumes [S1, S2].
3.  **Recursive Merging:** For each subsequent level $k > 1$, group eight adjacent texture maps from level $k-1$ [S1, S2]. Create one new texture map for level $k$ that contains the merged data of these eight parent maps [S1, S2]. The number of bricks sharing a single texture map is multiplied by a factor of 8 at each step [S1, S2].
4.  **Texture Coordinate Calculation:** When rendering a brick at level $k$, compute texture coordinates that map the brick's local voxel space to its assigned sub-region within the shared parent texture map [S1, S2].
5.  **Boundary Handling:** Ensure that boundary voxels are shared by adjacent bricks to prevent interpolation artifacts at boundaries between bricks at the same level [S1, S2]. Note that artifacts may still occur at boundaries between bricks on *different* levels, requiring additional correction techniques (see Subtopics) [S1, S2].

**Parameters**
*   **Number of Levels ($l$):** The depth of the hierarchy [S1, S2].
*   **Brick Count:** The starting number of bricks in every direction should match a power of two ($2^n$) to allow effective use of the hierarchy [S1, S2].

**Edge Cases and Pitfalls**
*   **Non-Power-of-Two Bricks:** If the number of bricks created by subdivision exceeds $2^l - 1$ (where $l$ is the number of levels), additional bricks are generated that build their own set of textures, bypassing the hierarchy [S1, S2].
*   **Fewer Bricks:** If fewer bricks are generated than required for a full hierarchy, the depth of the texture hierarchy must be reduced by storing textures on the coarsest level individually [S1, S2].
*   **Memory Overhead:** There is no saving of texture memory when switching from level 0 to level 1 because only half the voxels of every texture (plus one or two for overlap) contain non-redundant information [S1, S2]. Savings accumulate at higher levels [S1, S2].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class TextureManager:
    def __init__(self, num_bricks_per_dim, levels):
        self.num_bricks_per_dim = num_bricks_per_dim
        self.levels = levels
        self.texture_hierarchy = {}  # Maps (level, brick_id) -> texture_id

    def build_hierarchy(self, level_0_bricks):
        # Level 0: Each brick gets its own texture
        current_bricks = level_0_bricks
        current_level = 0
        for brick in current_bricks:
            # Sources do not specify GPU texture allocation and upload
            tex_id = self._allocate_texture(brick['data'])
            self.texture_hierarchy[(current_level, brick['id'])] = tex_id

        # Build higher levels
        for level in range(1, self.levels):
            next_bricks = []
            for i in range(0, len(current_bricks), 8):
                group = current_bricks[i:i+8]
                if len(group) < 8:
                    # Edge case: fewer bricks than expected for a full group
                    for brick in group:
                        prev_tex = self.texture_hierarchy.get((level-1, brick['id']))
                        self.texture_hierarchy[(level, brick['id'])] = prev_tex
                    break

                # Merge 8 bricks into one texture
                # Sources do not specify the exact packing/interpolation algorithm
                merged_data = self._pack_bricks(group)
                merged_tex = self._allocate_texture(merged_data)

                # Assign merged texture to all 8 bricks
                for brick in group:
                    self.texture_hierarchy[(level, brick['id'])] = merged_tex

                # Create parent brick for next iteration
                parent = {
                    'id': f"level_{level}_group_{i//8}",
                    'data': merged_data,
                    'origin': group[0]['origin'],
                    'children': group
                }
                next_bricks.append(parent)
            current_bricks = next_bricks
            current_level = level

    def compute_texture_coordinates(self, brick_origin, brick_size, level):
        # Returns (scale, offset) for texture sampling
        # At level k, brick size is 2^k * base_size.
        # Texture coordinates must scale by 1/2^k to map to the shared texture.
        scale = 1.0 / (2 ** level)
        offset = brick_origin * scale
        return scale, offset
```

## Subtopics

### [Opacity Correction by Polygon Clipping](5288-opacity-correction-by-polygon-clipping.md)
A technique to correct erroneously drawn pixels at boundaries between different resolution levels by projecting cutting planes along slice boundaries and using polygon clipping to determine the region of error.

## Operating characteristics
*   **Memory Efficiency:** Reduces texture memory overhead by merging adjacent bricks [S1, S2]. However, there is no saving when switching from level 0 to level 1 because only half the voxels (plus overlap) contain non-redundant information [S1, S2]. Savings increase at higher levels [S1, S2].
*   **Complexity:** The hierarchy depth is limited by the number of levels $l$. The number of bricks sharing a texture grows by a factor of 8 per level [S1, S2].
*   **Limitations:** Requires the number of bricks in each direction to be a power of two for optimal effectiveness [S1, S2]. Non-power-of-two brick counts lead to inefficient texture usage or reduced hierarchy depth [S1, S2].
*   **Artifacts:** Interpolation artifacts do not appear at boundaries between adjacent bricks at the same level due to shared boundary voxels [S1, S2]. However, artifacts can occur at boundaries between bricks on different levels, requiring additional correction [S1, S2].

## Background
Level-wise texture merging was introduced in the context of Level-of-Detail (LOD) volume rendering via 3D textures [S1, S2]. The method was proposed to address the high memory overhead associated with managing many small 3D textures for hierarchical volume rendering [S1, S2].

## Sources
[S1] Level-Of-Detail Volume Rendering via 3D Textures, https://publications.sci.utah.edu/publications/weiler00/IEEE-VolVis-2000.pdf, fetched 2026-10-27
[S2] Level-of-Detail Volume Rendering via 3D Textures, https://www.semanticscholar.org/paper/05e7d88387ac7484397e329456535b0c8c665a16, fetched 2026-11-03
