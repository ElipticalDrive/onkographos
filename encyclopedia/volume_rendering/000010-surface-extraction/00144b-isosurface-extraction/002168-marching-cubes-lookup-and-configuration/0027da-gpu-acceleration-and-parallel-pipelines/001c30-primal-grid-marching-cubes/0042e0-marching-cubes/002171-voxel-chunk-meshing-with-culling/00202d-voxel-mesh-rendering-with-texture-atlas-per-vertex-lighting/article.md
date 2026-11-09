# Voxel Mesh Rendering with Texture Atlas & Per-Vertex Lighting

Also called: Chunk Mesh Rendering, Voxel Meshing with Atlas UVs

## Overview
This method renders voxel chunks as triangle meshes by mapping block faces to a horizontal texture atlas and applying per-vertex lighting in the fragment shader [S1, unofficial]. It builds on [Voxel Chunk Meshing with Culling](2523-voxel-chunk-meshing-with-culling.md) to generate geometry, then optimizes rendering by batching materials into a single atlas and computing lighting values (ambient occlusion and smooth light) at the vertex level [S1, unofficial]. Unlike [Naive Voxel Meshing](2376-naive-voxel-meshing.md) which may use separate textures or per-face lighting, this approach reduces texture bind calls and supports complex block shapes (slabs, stairs) via specific UV offsets [S1, unofficial].

## How it works

### Inputs and Outputs
*   **Input**: A voxel chunk containing blocks with defined shapes (e.g., `WholeBlock`, `Slab`, `Stair`, `Ramp`, `Facade`) and culling modes (`Opaque`, `TransparentMerged`, `TransparentUnmerged`, `Empty`) [S1, unofficial].
*   **Input**: A set of 16x16 tile images for each block type [S1, unofficial].
*   **Output**: A vertex buffer containing positions, UVs, normals, lighting attributes, and block metadata [S1, unofficial].
*   **Output**: A 2D texture atlas (`Buffer2d`) containing all block face tiles concatenated horizontally [S1, unofficial].

### Data Structures

**Texture Atlas**
*   **Layout**: A 2D buffer of width `tiles.len() * 16` and height `16` [S1, unofficial].
*   **Content**: 16x16 pixel tiles concatenated horizontally. Each tile corresponds to a specific block face orientation [S1, unofficial].

**Vertex Structure**
Each vertex in the output mesh contains the following fields [S1, unofficial]:
*   `pos`: `Vec4<f32>` (x, y, z, 1.0)
*   `uv`: `Vec2<f32>` (local texture coordinates within the 16x16 tile)
*   `normal`: `Vec3<f32>` (face normal)
*   `ao`: `f32` (ambient occlusion, normalized)
*   `smooth_light`: `f32` (smooth light value, normalized)
*   `block`: `MyBlock` (enum identifying the block type)
*   `atlas_offset`: `f32` (U-offset in the atlas strip, representing the tile index)
*   `two_sided`: `bool` (flag for backface culling behavior)

**Interpolated Output (VsOut)**
Attributes passed to the fragment shader [S1, unofficial]:
*   `uv`: `Vec2<f32>`
*   `normal`: `Vec3<f32>`
*   `atlas_u_offset`: `f32`
*   `ao`: `f32`
*   `smooth_light`: `f32`

### Steps

1.  **Build Texture Atlas**: Load all 16x16 tile images for the block types and concatenate them horizontally into a `Buffer2d` of width `N * 16` and height `16`, where `N` is the number of unique block faces/tiles [S1, unofficial].
2.  **Map Block Types to Atlas Offsets**: For each block type and face orientation, calculate a U-offset (`atlas_offset`) in the range `[0.0, 14.0]` representing the tile index [S1, unofficial]. Note that some blocks (e.g., Cactus) use different offsets for top/bottom faces versus side faces [S1, unofficial].
3.  **Generate Mesh Geometry**: Iterate through the voxel chunk (as defined in [Voxel Chunk Meshing with Culling](2523-voxel-chunk-meshing-with-culling.md)) to identify exposed faces [S1, unofficial]. For each exposed face (quad), determine the block type and shape (e.g., `WholeBlock`, `Stair`, `Ramp`) [S1, unofficial]. Calculate the `atlas_offset` based on block type and face orientation [S1, unofficial]. Determine if the geometry is `two_sided` (true if the face is diagonal or if the shape is a `Facade`) [S1, unofficial]. Compute vertex attributes for the four corners of the quad: `pos` (world position), `uv` (local 0.0–1.0), `normal` (face normal or slope normal for diagonals), `ao` (quad AO divided by 3.0), `smooth_light` (quad light divided by 15.0), `block` (type enum), `atlas_offset` (pre-calculated U-offset), and `two_sided` (boolean flag) [S1, unofficial]. Use anisotropy-corrected indices to ensure proper interpolation of AO and lighting across the quad [S1, unofficial].
4.  **Render Pipeline**: Render opaque geometry first with backface culling enabled, excluding `two_sided` vertices [S1, unofficial]. Render two-sided geometry next without backface culling, including `two_sided` vertices (cross-shaped billboards, facades, diagonal slopes) [S1, unofficial]. Render transparent geometry last if applicable [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_mesh_and_render(chunk, atlas_tiles):
    # 1. Build texture atlas
    atlas_width = len(atlas_tiles) * 16
    atlas_height = 16
    atlas = Buffer2d(width=atlas_width, height=atlas_height)
    for i, tile in enumerate(atlas_tiles):
        atlas.copy_tile(tile, u_offset=i * 16)

    # 2. Map block types to atlas offsets
    atlas_offsets = {}
    for block_type in chunk.block_types:
        for face in block_type.faces:
            atlas_offsets[(block_type, face)] = calculate_u_offset(block_type, face)

    # 3. Generate mesh geometry
    vertices = []
    for quad in chunk.exposed_quads:
        block = quad.block
        shape = quad.shape
        two_sided = quad.is_diagonal() or shape == Shape.Facade
        atlas_off = atlas_offsets[(block, quad.face)]
        normal = quad.slope_normal if two_sided else quad.face_normal

        for i in range(4):
            vertices.append(Vertex(
                pos=quad.positions[i],
                uv=quad.uvs[i],
                normal=normal,
                ao=quad.ao[i] / 3.0,
                smooth_light=quad.light[i] / 15.0,
                block=block,
                atlas_offset=atlas_off,
                two_sided=two_sided
            ))

    # 4. Render pipeline
    render_opaque(vertices, cull_mode=BACKFACE)
    render_two_sided(vertices, cull_mode=NONE)
    render_transparent(vertices, cull_mode=NONE)
```

### Formulas

**Lighting Calculation**
The final color shade is computed in the fragment shader using the interpolated vertex attributes [S1, unofficial]:

$$
shade = (ambient + (1.0 - ambient) \cdot \max(0.0, \mathbf{n} \cdot \mathbf{l})) \cdot ao + smooth\_light \cdot 0.6
$$

Where:
*   $ambient = 0.35$ (base illumination factor) [S1, unofficial]
*   $\mathbf{l} = \text{normalize}(0.4, 0.7, 0.5)$ (directional light vector) [S1, unofficial]
*   $\mathbf{n}$ is the interpolated vertex normal [S1, unofficial]
*   $ao \in [0, 1]$ is the interpolated ambient occlusion [S1, unofficial]
*   $smooth\_light \in [0, 1]$ is the interpolated smooth light value [S1, unofficial]

**Atlas UV Lookup**
To handle wrapping and clamping in the fragment shader [S1, unofficial]:
*   The U-coordinate is remapped using `rem_euclid(1.0)` to handle wrapping [S1, unofficial].
*   The U-coordinate is clamped to `min(atlas_size[0] - 1)` to prevent out-of-bounds access [S1, unofficial].

**Alpha Blending**
*   A threshold of `0.01` is used to skip blending for near-zero alpha values, preventing artifacts [S1, unofficial].

### Parameters

*   `ambient`: 0.35 (base illumination factor) [S1, unofficial]
*   `light_dir`: (0.4, 0.7, 0.5) (normalized directional light vector) [S1, unofficial]
*   `smooth_light_weight`: 0.6 (multiplier for smooth light contribution) [S1, unofficial]
*   `tile_size`: 16 (width and height of each block face texture in the atlas) [S1, unofficial]

### Edge Cases and Pitfalls

*   **Diagonal Faces**: Stairs and ramps require specific slope normals and two-sided rendering to avoid visual artifacts. The normal is calculated as the normalized sum of the slope face normal and the front face normal, scaled by $1/\sqrt{2}$ [S1, unofficial].
*   **Atlas Wrapping**: UV lookups must handle edge cases where the U-coordinate wraps around the atlas width. Using `rem_euclid` and clamping ensures correct sampling [S1, unofficial].
*   **AO Interpolation**: Standard vertex indices may cause incorrect AO interpolation across quads. Anisotropy-corrected indices must be used to ensure smooth lighting transitions [S1, unofficial].
*   **Transparent Blocks**: Blocks with `CullMode::TransparentMerged` (e.g., glass) only cull faces between identical block types. Blocks with `CullMode::TransparentUnmerged` (e.g., leaves) always render faces, even between identical blocks [S1, unofficial].

## Operating characteristics

*   **Time Complexity**: Linear with respect to the number of exposed faces in the chunk. Atlas construction is constant relative to chunk size but depends on the number of unique block types [S1, unofficial].
*   **Memory Complexity**: Linear with respect to the number of vertices in the mesh. The atlas size is constant relative to chunk size but depends on the number of unique block textures [S1, unofficial].
*   **Performance**: High performance due to reduced texture bind calls (single atlas) and efficient vertex lighting. Suitable for real-time rendering of large voxel worlds [S1, unofficial].
*   **Strengths**: Efficient batching, support for complex block shapes, smooth lighting via per-vertex AO [S1, unofficial].
*   **Limitations**: Limited to 16x16 tile resolution per block face. Atlas size grows with the number of unique block types. Two-sided rendering increases draw calls or requires separate pipelines [S1, unofficial].

## Background

This method is implemented in the `voxmesh` repository (referred to as `Cactus` in the source code) [S1, unofficial]. It is commonly used in voxel-based games and simulations where performance and visual quality are balanced through efficient mesh generation and lighting techniques [S1, unofficial].

## Sources

[S1] finnbear/voxmesh, https://github.com/finnbear/voxmesh, fetched 2026-10-26 (unofficial)
