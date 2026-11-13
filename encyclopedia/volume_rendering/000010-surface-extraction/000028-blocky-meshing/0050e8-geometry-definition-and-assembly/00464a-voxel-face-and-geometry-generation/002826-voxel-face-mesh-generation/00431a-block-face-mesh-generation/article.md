# Block Face Mesh Generation

Also called: addFace

## Overview
Block Face Mesh Generation generates a 6-vertex mesh data structure for a single exposed face of a voxel block, applying special geometry for cactus blocks and packing vertex attributes for GPU consumption [S1, unofficial]. It differs from its sibling [Face Direction and Quad Geometry](4881-face-direction-and-quad-geometry.md) by handling block-specific geometric modifications (like cactus insets) and producing a packed vertex format rather than just raw geometric primitives [S1, S2, unofficial]. It serves as the core geometry generation step within [Voxel Face Mesh Generation](4212-voxel-face-mesh-generation.md) [S1, unofficial].

## How it works
**Inputs:**
- `face`: Integer index of the face to generate (0-5).
- `faceX`, `faceY`, `faceZ`: Chunk-local coordinates of the block.
- `blockID`: Integer identifier of the block type.
- `TextureManager`: Object providing texture layer information.
- `faceData`: Array mapping face indices to vertex offsets.
- `unitVertices`: Array of 8 unit cube vertices.
- `faceVertices`: Array mapping face indices to vertex indices within the unit cube.
- `blockToVecMapping`: Array mapping face indices to normal vectors.
- `cactusInset`: Float value for cactus geometry adjustment (typically $1/16$).

**Outputs:**
- `vertices`: Vector of packed vertex structures.
- `indices`: Vector of index integers forming two triangles.

**Data Structures:**
- `packed_vertex`: A struct containing 9 fields packed into a single integer/struct representation for GPU consumption [S1, unofficial]. Fields include:
  - `position`: 3 floats ($px, py, pz$).
  - `normal_index`: `uint32_t` representing the face direction.
  - `corner_index`: `uint32_t` representing the corner position.
  - `texture_layer`: `uint32_t` from `TextureManager`.
  - `sky_light_level`: `float`.
  - `water_above`: `bool`.
  - `block_light_level`: `float` [S1, unofficial].
- `unitVertices[8]`: Contains 8 `glm::vec3` coordinates for a unit cube: $(0,0,0), (1,0,0), (1,1,0), (0,1,0), (0,0,1), (1,0,1), (1,1,1), (0,1,1)$ [S2, unofficial].
- `faceVertices[6][4]`: Maps face indices to 4 vertex indices per face:
  - TOP (2): $\{3, 2, 6, 7\}$
  - BOTTOM (3): $\{1, 0, 4, 5\}$
  - LEFT (4): $\{4, 0, 3, 7\}$
  - RIGHT (5): $\{1, 5, 6, 2\}$
  - FRONT (0): $\{0, 1, 2, 3\}$
  - BACK (1): $\{5, 4, 7, 6\}$ [S2, unofficial].
- `BlockTexture`: Struct containing `glm::vec2` for `topTexture`, `sideTexture`, and `bottomTexture` [S2, unofficial].

**Parameters:**
- `CANVAS_SIZE = 16`: Defines texture atlas grid dimension [S2, unofficial].
- `CELL_SIZE = 1.0f / CANVAS_SIZE`: UV step size per texture tile [S2, unofficial].
- `BLOCK_AIR = 255`: Block ID that skips mesh generation [S2, unofficial].

**Steps:**
1. **Air Check:** If `blockID` equals `BLOCK_AIR` (255), return immediately without generating geometry [S2, unofficial].
2. **Base Position Calculation:** For each of the 6 vertices ($i=0..5$), compute the world-local position using `faceData`:
   $$
   \begin{aligned}
   px &= faceX + faceData[face][i \times 3 + 0] \\
   py &= faceY + faceData[face][i \times 3 + 1] \\
   pz &= faceZ + faceData[face][i \times 3 + 2]
   \end{aligned}
   $$
   [S1, unofficial].
3. **Cactus Inset:** If `blockID` is CACTUS and the face is not TOP (2) or BOTTOM (3), apply inset adjustments:
   - Face 0 (Z+): $pz = faceZ + 1.0f - cactusInset$
   - Face 1 (Z-): $pz = faceZ + cactusInset$
   - Face 4 (X+): $px = faceX + 1.0f - cactusInset$
   - Face 5 (X-): $px = faceX + cactusInset$
   [S1, unofficial].
4. **Texture Offset Determination:** Determine texture offset based on face index: 0 for TOP, 1 for BOTTOM, else SIDE [S2, unofficial].
5. **Vertex Packing:** Pack vertex data into `packed_vertex` using position ($px, py, pz$), normal index (face), corner index (`packed_vertex::CORNER_FOR_VERT[i]`), texture layer (from `TextureManager`), sky light level, water above flag, and block light level [S1, unofficial].
6. **UV Calculation:** Calculate 4 UV coordinates for the face quad:
   $$
   \begin{aligned}
   uv_0 &= (offset + (0, CELL_SIZE)) \\
   uv_1 &= (offset + (CELL_SIZE, CELL_SIZE)) \\
   uv_2 &= (offset + (CELL_SIZE, 0)) \\
   uv_3 &= (offset + (0, 0))
   \end{aligned}
   $$
   [S2, unofficial].
7. **Vertex Generation:** For each of the 4 vertices in the face quad ($vIndex$):
   - Compute world position by adding `unitVertices[vIndex]` to the base position.
   - Set vertex normal from `blockToVecMapping[face]`.
   - Set texture coordinates from the precomputed UV array.
   - Push to `vertices` vector [S2, unofficial].
8. **Index Generation:** Record `indexOffset = vertices.size()`, then append 6 indices forming two triangles: $(offset, offset+1, offset+2, offset, offset+2, offset+3)$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_block_face_mesh(face, faceX, faceY, faceZ, blockID, faceData, unitVertices, faceVertices, blockToVecMapping, texture_manager, cactus_inset=1.0/16.0):
    # Step 1: Air Check
    if blockID == 255:
        return [], []

    vertices = []
    indices = []

    # Step 2 & 3: Base Position and Cactus Inset
    # Note: faceData layout is assumed to be [face][vertex_index * 3 + component]
    # We compute positions for the 6 vertices of the face geometry
    positions = []
    for i in range(6):
        px = faceX + faceData[face][i * 3 + 0]
        py = faceY + faceData[face][i * 3 + 1]
        pz = faceZ + faceData[face][i * 3 + 2]
        
        # Cactus Inset Logic
        if blockID == CACTUS_ID and face not in [TOP, BOTTOM]:
            if face == 0: # Z+
                pz = faceZ + 1.0 - cactus_inset
            elif face == 1: # Z-
                pz = faceZ + cactus_inset
            elif face == 4: # X+
                px = faceX + 1.0 - cactus_inset
            elif face == 5: # X-
                px = faceX + cactus_inset
        
        positions.append((px, py, pz))

    # Step 4: Texture Offset
    if face == TOP:
        texture_offset = 0
    elif face == BOTTOM:
        texture_offset = 1
    else:
        texture_offset = SIDE_OFFSET # Implementation detail for SIDE

    # Step 6: UV Calculation
    cell_size = 1.0 / 16.0
    uvs = [
        (texture_offset + (0, cell_size)),
        (texture_offset + (cell_size, cell_size)),
        (texture_offset + (cell_size, 0)),
        (texture_offset + (0, 0))
    ]

    # Step 5 & 7: Vertex Generation
    # faceVertices maps face index to 4 vertex indices in unitVertices
    face_vertex_indices = faceVertices[face]
    normal = blockToVecMapping[face]
    
    # Determine which of the 6 positions correspond to the 4 quad vertices
    # The mapping from faceVertices to positions depends on the specific faceData layout.
    # Assuming faceVertices indices correspond to the order in positions for the quad vertices.
    # Note: faceVertices contains indices into unitVertices, which define the quad shape.
    # The positions array has 6 entries. We need to select the 4 relevant ones.
    # This selection logic is implicit in the source's faceVertices mapping relative to faceData.
    # For implementation, we assume faceVertices[i] gives the index into the positions array 
    # for the i-th vertex of the quad.
    
    quad_positions = [positions[face_vertex_indices[i]] for i in range(4)]
    
    for i in range(4):
        pos = quad_positions[i]
        uv = uvs[i]
        
        # Packed vertex creation
        # CORNER_FOR_VERT is a mapping from vertex index to corner index
        corner_idx = CORNER_FOR_VERT[i] 
        tex_layer = texture_manager.get_layer(blockID, face)
        
        packed_v = {
            'position': pos,
            'normal_index': face,
            'corner_index': corner_idx,
            'texture_layer': tex_layer,
            'sky_light_level': get_sky_light(faceX, faceY, faceZ), # Placeholder for lighting logic
            'water_above': check_water_above(faceX, faceY, faceZ), # Placeholder
            'block_light_level': get_block_light(faceX, faceY, faceZ) # Placeholder
        }
        vertices.append(packed_v)

    # Step 8: Index Generation
    index_offset = len(vertices) - 4
    indices.extend([
        index_offset, index_offset + 1, index_offset + 2,
        index_offset, index_offset + 2, index_offset + 3
    ])

    return vertices, indices
```

**Edge Cases and Pitfalls:**
- **Air Blocks:** Must be checked before geometry generation to avoid unnecessary processing [S2, unofficial].
- **Cactus Geometry:** Only applies to side faces; top and bottom faces remain full cube [S1, unofficial].
- **Precision:** Floating point positions should be calculated carefully to avoid z-fighting with adjacent faces [G].
- **Face Data Layout:** The exact mapping of `faceData` indices to vertex positions must match the `faceVertices` mapping to ensure correct quad formation [S1, S2, unofficial].

## Subtopics

### [Water Face Mesh Generation](10936-water-face-mesh-generation.md)
Generates mesh vertices for water blocks using the same geometric layout as terrain faces but omits texture layer and block light to maintain a uniform vertex format while reducing shader workload.

## Operating characteristics
- **Time Complexity:** $O(1)$ per face, as it generates a fixed number of vertices (6) and indices (6) [S1, S2, unofficial].
- **Memory Complexity:** $O(1)$ per face, storing a small fixed-size vertex and index buffer [S1, S2, unofficial].
- **Performance:** Efficient for real-time rendering due to minimal per-face overhead [S1, unofficial].
- **Strengths:** Simple geometry generation, easy to integrate with texture atlases [S2, unofficial].
- **Limitations:** Does not handle complex block shapes beyond cactus insets [S1, unofficial].
- **When to Use:** For voxel engines requiring simple block geometry with minimal per-face processing [S1, S2, unofficial].
- **When Not to Use:** For blocks requiring complex geometry or smooth shading [S1, S2, unofficial].

## Background
Block Face Mesh Generation is a method used in voxel engine rendering pipelines, specifically in implementations like `ft_minecraft` and `VoxelEngine` [S1, S2, unofficial]. It evolved from the need to efficiently render voxel-based worlds by generating mesh data for individual block faces, allowing for culling and optimization of visible surfaces [S1, S2, unofficial].

## Sources
[S1] ldominiq/ft_minecraft, https://github.com/ldominiq/ft_minecraft, fetched 2026-10-27 (unofficial)
[S2] GitHub Repo for VoxelEngine, https://github.com/TheSlabby/VoxelEngine, fetched 2026-10-29 (unofficial)
