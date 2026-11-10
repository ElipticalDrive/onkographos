# Block Render Data Structure & Instance Generation

Also called: BlockRenderData, Face Tint Mixing

## Overview
This method defines the data structure for individual voxel blocks and the process of generating GPU instances for rendering. It solves the problem of efficiently passing per-face texture and color information to the graphics API by packing block-specific attributes into a single struct. It differs from sibling methods by focusing on the atomic unit of rendering data (the block) rather than the assembly of chunks or the final mesh submission.

## How it works

### Inputs and Outputs
- **Inputs**: Texture name strings, texture IDs (integers), face tint colors (3-component vectors), and a shape reference.
- **Outputs**: A `BlockRenderData` struct instance containing packed face data.

### Data Structures
The core data structure is `BlockRenderData`. It contains 13 fields:
1.  `faceBack`: `int` (Texture ID)
2.  `faceFront`: `int` (Texture ID)
3.  `faceTop`: `int` (Texture ID)
4.  `faceBottom`: `int` (Texture ID)
5.  `faceLeft`: `int` (Texture ID)
6.  `faceRight`: `int` (Texture ID)
7.  `tintBack`: `Vector3` (RGB tint)
8.  `tintFront`: `Vector3` (RGB tint)
9.  `tintTop`: `Vector3` (RGB tint)
10. `tintBottom`: `Vector3` (RGB tint)
11. `tintLeft`: `Vector3` (RGB tint)
12. `tintRight`: `Vector3` (RGB tint)
13. `shape`: `IShape` reference

### Algorithm Steps

1.  **Texture ID Lookup**: Convert texture names to integer IDs using a mapping dictionary. If a name is not found, the behavior is undefined in the sources, but typically implies a default or error [S1, unofficial].
2.  **Tint Initialization**: If no specific tints are provided, default all face tints to white (`Vector3(1, 1, 1)`) [S1, unofficial].
3.  **Data Packing**: Populate the `BlockRenderData` struct with the 6 texture IDs and 6 tint vectors corresponding to the 6 faces of the voxel.
4.  **Instance Generation**: Pass the populated `BlockRenderData` to the shape's instance generation method. The shape uses this data to determine which texture atlas index and color tint to apply to each visible face during rendering [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
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

def create_block_render_data(texture_id_map, face_back_name, face_front_name, face_top_name, face_bottom_name, face_left_name, face_right_name,
                             tint_back=None, tint_front=None, tint_top=None, tint_bottom=None, tint_left=None, tint_right=None,
                             shape=None):
    """
    Creates a BlockRenderData instance.
    texture_id_map: Dict[str, int] mapping texture names to IDs.
    tint_*: Optional Vector3. If None, defaults to (1, 1, 1).
    """
    white = Vector3(1.0, 1.0, 1.0)
    
    # Default tints to white if not provided
    if tint_back is None: tint_back = white
    if tint_front is None: tint_front = white
    if tint_top is None: tint_top = white
    if tint_bottom is None: tint_bottom = white
    if tint_left is None: tint_left = white
    if tint_right is None: tint_right = white

    # Lookup texture IDs
    face_back_id = texture_id_map[face_back_name]
    face_front_id = texture_id_map[face_front_name]
    face_top_id = texture_id_map[face_top_name]
    face_bottom_id = texture_id_map[face_bottom_name]
    face_left_id = texture_id_map[face_left_name]
    face_right_id = texture_id_map[face_right_name]

    return BlockRenderData(face_back_id, face_front_id, face_top_id, face_bottom_id, face_left_id, face_right_id,
                           tint_back, tint_front, tint_top, tint_bottom, tint_left, tint_right, shape)

def generate_instances(block_data_list, shape):
    """
    Iterates over a list of BlockRenderData and generates instances.
    """
    for data in block_data_list:
        shape.instance(data)
```

### Parameters and Edge Cases
- **Default Tint**: White (`1, 1, 1`) is the standard default [S1, unofficial].
- **Shape Reference**: The `IShape` object is stored in the struct, linking the data to the rendering primitive [S1, unofficial].
- **Missing Textures**: The sources do not specify behavior for missing texture names in the map.

## Operating characteristics
- **Memory**: Each block instance carries 6 integers and 6 vectors. This is relatively heavy per-vertex/per-instance data compared to simple color-only blocks, but necessary for per-face texturing.
- **Performance**: Instance generation is O(N) where N is the number of visible blocks. The lookup of texture IDs is O(1) with a hash map.
- **Strengths**: Allows fine-grained control over per-face appearance.
- **Limitations**: High memory bandwidth usage for instance attributes.

## Background
This method originates from the `Cubic-Grid_Voxel_Renderer` project by rarefushion [S1, unofficial]. It is part of a voxel rendering pipeline that uses instanced rendering to draw voxel grids.

## Sources
[S1] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
