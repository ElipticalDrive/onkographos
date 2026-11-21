# Block Render Data Structure & Instance Generation

Also called: BlockRenderData, Face Tint Mixing

## Overview
This method aggregates per-face rendering metadata (texture IDs and color tints) into a compact data structure to drive instanced mesh generation for voxel blocks [S1, unofficial]. It maps logical face attributes to GPU instancing parameters without duplicating geometry data [S1, unofficial]. It differs from sibling methods by focusing on the attribute assembly phase rather than geometric mesh construction, preparing data for the subsequent mesh assembly stage [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
- A mapping from texture names to integer IDs (`textureIDByName`) [S1, unofficial].
- Face texture names (strings) for each of the 6 block faces: Back, Front, Top, Bottom, Left, Right [S1, unofficial].
- Face tint colors (`Vector3`) for each of the 6 faces. Defaults to white `(1, 1, 1)` if not specified [S1, unofficial].
- A reference to the geometric shape (`IShape`) to be instanced [S1, unofficial].

**Outputs:**
- A `BlockRenderData` struct instance containing the resolved texture IDs, tints, and shape reference [S1, unofficial].

### Data Structures
The core data structure is `BlockRenderData` [S1, unofficial]. It is a struct containing 13 fields:
1. `faceBack`: `int` (Texture ID)
2. `faceFront`: `int` (Texture ID)
3. `faceTop`: `int` (Texture ID)
4. `faceBottom`: `int` (Texture ID)
5. `faceLeft`: `int` (Texture ID)
6. `faceRight`: `int` (Texture ID)
7. `tintBack`: `Vector3` (RGB Tint)
8. `tintFront`: `Vector3` (RGB Tint)
9. `tintTop`: `Vector3` (RGB Tint)
10. `tintBottom`: `Vector3` (RGB Tint)
11. `tintLeft`: `Vector3` (RGB Tint)
12. `tintRight`: `Vector3` (RGB Tint)
13. `shape`: `IShape` (Reference to the mesh geometry) [S1, unofficial].

### Steps
1. **Initialize Texture Map:** Create a dictionary `textureIDByName` that maps texture name strings to unique integer IDs, typically derived from a `TextureLoader.Texture[]` array where the index serves as the ID [S1, unofficial].
2. **Resolve Texture IDs:** For each face (Back, Front, Top, Bottom, Left, Right), look up the provided texture name string in `textureIDByName` to retrieve the corresponding integer ID [S1, unofficial].
3. **Prepare Tints:** Assign the provided `Vector3` tint for each face. If no tint is provided for a specific face, default to `Vector3.One` (white) [S1, unofficial].
4. **Construct Struct:** Instantiate `BlockRenderData` with the 6 resolved integer IDs, the 6 `Vector3` tints, and the `IShape` reference [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Vector3:
    def __init__(self, x, y, z):
        self.x, self.y, self.z = x, y, z

class BlockRenderData:
    def __init__(self, face_back_id, face_front_id, face_top_id, face_bottom_id, face_left_id, face_right_id,
                 tint_back, tint_front, tint_top, tint_bottom, tint_left, tint_right, shape):
        self.face_back = face_back_id
        self.face_front = face_front_id
        self.face_top = face_top_id
        self.face_bottom = face_bottom_id
        self.face_left = face_left_id
        self.face_right = face_right_id
        self.tint_back = tint_back
        self.tint_front = tint_front
        self.tint_top = tint_top
        self.tint_bottom = tint_bottom
        self.tint_left = tint_left
        self.tint_right = tint_right
        self.shape = shape

def create_block_render_data(texture_name_to_id_map, face_names, face_tints, shape):
    WHITE = Vector3(1.0, 1.0, 1.0)
    face_back_id = texture_name_to_id_map[face_names[0]]
    face_front_id = texture_name_to_id_map[face_names[1]]
    face_top_id = texture_name_to_id_map[face_names[2]]
    face_bottom_id = texture_name_to_id_map[face_names[3]]
    face_left_id = texture_name_to_id_map[face_names[4]]
    face_right_id = texture_name_to_id_map[face_names[5]]
    if face_tints is None:
        face_tints = (WHITE, WHITE, WHITE, WHITE, WHITE, WHITE)
    return BlockRenderData(
        face_back_id, face_front_id, face_top_id, face_bottom_id, face_left_id, face_right_id,
        face_tints[0], face_tints[1], face_tints[2], face_tints[3], face_tints[4], face_tints[5],
        shape
    )
```

### Parameters
- **Default Tint:** `Vector3(1, 1, 1)` (White) [S1, unofficial].
- **Texture ID Source:** Index in the `TextureLoader.Texture[]` array [S1, unofficial].

### Edge Cases and Pitfalls
- **Missing Texture Names:** If a texture name provided in `face_names` is not present in `textureIDByName`, the lookup will fail. The source does not specify a fallback [G].
- **Tint Mixing:** The notes mention "Face Tint Mixing" and "Mix provided face tints with base block tints by element-wise multiplication" or "Replace base block tints entirely" [S1, unofficial]. The primary code structure uses provided tints directly. Mixing logic is not implemented within the struct constructor itself [S1, unofficial].

## Subtopics

### [Chunk Mesh Assembly](4100-chunk-mesh-assembly.md)
Aggregates `BlockRenderData` instances into contiguous buffers. It sets itself apart by handling memory layout and GPU buffer uploads rather than attribute resolution [G]. Choose it when batching multiple blocks for draw calls [G].

### [Mesh Plugin Architecture](4921-mesh-plugin-architecture.md)
Defines how render data is consumed by different rendering backends. It sets itself apart by abstracting backend-specific rendering APIs [G]. Choose it when supporting multiple graphics APIs [G].

### [Material-Based Mesh Sectioning](8778-material-based-mesh-sectioning.md)
Groups blocks by material before instance generation. It sets itself apart by sorting geometry by shader requirements rather than raw attributes [G]. Choose it when minimizing state changes [G].

### [Mesh Finalization](11478-mesh-finalization.md)
Finalizes the GPU buffers created from the assembled data. It sets itself apart by handling buffer binding and draw call submission [G]. Choose it as the last step before rendering [G].

## Operating characteristics
- **Time Complexity:** $O(1)$ per block for data structure creation, assuming dictionary lookup is $O(1)$ [S1, unofficial].
- **Memory Complexity:** $O(1)$ per block for the `BlockRenderData` struct. The struct size is fixed (6 ints + 6 vectors + 1 reference) [S1, unofficial].
- **Strengths:** Compact representation of per-face attributes; separates attribute data from geometry data [S1, unofficial].
- **Limitations:** Does not handle geometry generation; relies on downstream assembly for batching [S1, unofficial].

## Background
Implemented in the referenced repository [S1, unofficial]. It is part of a voxel rendering pipeline that uses instanced rendering for efficiency [S1, unofficial].

## Sources
[S1] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
