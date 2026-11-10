# Block Render Data Structure & Instance Generation

Also called: BlockRenderData, Face Tint Mixing

## Overview
This method defines the data structure and generation logic for translating discrete voxel face attributes into GPU-drawable instances. It solves the problem of mapping per-face texture IDs and color tints to a unified instance buffer for efficient rendering. It differs from sibling methods by focusing on the specific aggregation of face-level metadata (textures and tints) into a single struct per block, rather than the broader mesh assembly or plugin architecture.

## How it works
**Inputs:**
- A list of face texture names (strings) for each of the 6 directions: Back, Front, Top, Bottom, Left, Right.
- A list of face tint colors (vectors) for each of the 6 directions.
- A reference to the block shape (`IShape`).
- A texture loader mapping texture names to IDs.

**Outputs:**
- A `BlockRenderData` struct instance containing resolved texture IDs, tints, and shape reference.

**Data Structures:**
The core structure is `BlockRenderData`. It contains:
- 6 integer fields for face texture IDs: `faceBack`, `faceFront`, `faceTop`, `faceBottom`, `faceLeft`, `faceRight`.
- 6 vector fields for face tints: `tintBack`, `tintFront`, `tintTop`, `tintBottom`, `tintLeft`, `tintRight`.
- 1 reference field: `shape` (of type `IShape`).

**Steps:**
1. **Texture ID Resolution:** Map each input face texture name to a unique integer ID using a lookup table (e.g., `FrozenDictionary<string, int>`). If a name is not found, handle as an error or default [S1, unofficial].
2. **Tint Handling:**
   - If no specific tints are provided, default all 6 face tints to white (`Vector3(1, 1, 1)`) [S1, unofficial].
   - If specific tints are provided, use them directly.
   - **Face Tint Mixing:** If base block tints and provided face tints both exist, mix them by element-wise multiplication before instance generation. Alternatively, replace base block tints entirely with provided face tints [S1, unofficial].
3. **Instance Generation:** Collect the resolved texture IDs and mixed tints for all visible faces. Pass these attributes to the shape's `Instance` method to generate the final render data [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class BlockRenderData:
    def __init__(self, face_back_id, face_front_id, face_top_id, face_bottom_id, face_left_id, face_right_id,
                 tint_back, tint_front, tint_top, tint_bottom, tint_left, tint_right, shape):
        self.face_back_id = face_back_id
        self.face_front_id = face_front_id
        self.face_top_id = face_top_id
        self.face_bottom_id = face_bottom_id
        self.face_left_id = face_left_id
        self.face_right_id = face_right_id
        self.tint_back = tint_back
        self.tint_front = tint_front
        self.tint_top = tint_top
        self.tint_bottom = tint_bottom
        self.tint_left = tint_left
        self.tint_right = tint_right
        self.shape = shape

def create_block_render_data(texture_name_by_id, texture_id_by_name, face_back_name, face_front_name, face_top_name,
                             face_bottom_name, face_left_name, face_right_name,
                             tint_back=None, tint_front=None, tint_top=None, tint_bottom=None, tint_left=None, tint_right=None,
                             shape=None):
    # Default tint is white
    white_tint = Vector3(1, 1, 1)
    
    # Resolve texture IDs
    face_back_id = texture_id_by_name[face_back_name]
    face_front_id = texture_id_by_name[face_front_name]
    face_top_id = texture_id_by_name[face_top_name]
    face_bottom_id = texture_id_by_name[face_bottom_name]
    face_left_id = texture_id_by_name[face_left_name]
    face_right_id = texture_id_by_name[face_right_name]
    
    # Handle tints
    if tint_back is None: tint_back = white_tint
    if tint_front is None: tint_front = white_tint
    if tint_top is None: tint_top = white_tint
    if tint_bottom is None: tint_bottom = white_tint
    if tint_left is None: tint_left = white_tint
    if tint_right is None: tint_right = white_tint
    
    # Mix tints if necessary (element-wise multiplication)
    # Note: Specific mixing logic depends on whether base tints are provided.
    # If only face tints are provided, they are used directly.
    
    return BlockRenderData(face_back_id, face_front_id, face_top_id, face_bottom_id, face_left_id, face_right_id,
                           tint_back, tint_front, tint_top, tint_bottom, tint_left, tint_right, shape)
```

**Parameters:**
- `texture_id_by_name`: Dictionary mapping texture names to integer IDs.
- `tint`: Vector3 representing color tint. Default is white.

**Edge Cases:**
- Missing texture names: Should raise an error or use a default texture [S1, unofficial].
- Null tints: Default to white [S1, unofficial].

## Subtopics

### [Chunk Mesh Assembly](4100-chunk-mesh-assembly.md)
Aggregates mesh data into chunks. Choose when optimizing for draw call reduction.

### [Mesh Plugin Architecture](4921-mesh-plugin-architecture.md)
Decouples meshing algorithms. Choose when supporting multiple meshing techniques.

### [Mesh Finalization](11478-mesh-finalization.md)
Finalizes mesh data for rendering. Choose when preparing data for GPU upload.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of faces, as each face requires texture ID lookup and tint processing.
- **Memory Complexity:** Constant per block, as the `BlockRenderData` struct has a fixed size.
- **Strengths:** Simple data structure, easy to implement, supports per-face customization.
- **Limitations:** May not be optimal for very large scenes due to per-block overhead.
- **When to use:** When per-face texture and tint customization is required.
- **When not to use:** When memory efficiency is critical and per-face customization is not needed.

## Background
Originated in the context of voxel-based rendering systems, specifically in the `Cubic-Grid_Voxel_Renderer` project [S1, unofficial]. It addresses the need for efficient mapping of voxel face attributes to GPU instances.

## Sources
[S1] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
