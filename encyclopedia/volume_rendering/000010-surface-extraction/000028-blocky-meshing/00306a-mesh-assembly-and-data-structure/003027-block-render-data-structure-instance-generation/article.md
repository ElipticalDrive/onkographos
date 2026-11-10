# Block Render Data Structure & Instance Generation

Also called: BlockRenderData, Face Tint Mixing

## Overview
This method defines the data structure and generation process for individual voxel instances within a blocky meshing pipeline. It solves the problem of associating specific visual properties (texture IDs and color tints) with each face of a voxel block, enabling per-face material variation and color modulation. It differs from sibling methods by focusing on the atomic unit of rendering data (the `BlockRenderData` struct) rather than the bulk assembly of vertex buffers or the final mesh finalization.

## How it works
### Inputs and Outputs
- **Inputs**:
  - A mapping of texture names to integer IDs (`FrozenDictionary<string, int>`).
  - A set of face texture names (strings) for Back, Front, Top, Bottom, Left, Right.
  - A set of face tint colors (`Vector3`), defaulting to white `(1, 1, 1)` if not specified.
  - A reference to the geometric shape (`IShape`) defining the voxel geometry.
- **Outputs**:
  - A `BlockRenderData` struct instance containing resolved texture IDs, tints, and shape reference.

### Data Structures
The core data structure is `BlockRenderData`. It is a struct containing the following fields:
- `faceBack`: `int` (Texture ID)
- `faceFront`: `int` (Texture ID)
- `faceTop`: `int` (Texture ID)
- `faceBottom`: `int` (Texture ID)
- `faceLeft`: `int` (Texture ID)
- `faceRight`: `int` (Texture ID)
- `tintBack`: `Vector3` (RGB tint)
- `tintFront`: `Vector3` (RGB tint)
- `tintTop`: `Vector3` (RGB tint)
- `tintBottom`: `Vector3` (RGB tint)
- `tintLeft`: `Vector3` (RGB tint)
- `tintRight`: `Vector3` (RGB tint)
- `shape`: `IShape` reference

### Steps
1. **Texture ID Resolution**: For each of the six faces, resolve the provided texture name string to an integer ID using the `textureIDByName` dictionary. If a name is not found, the behavior is undefined in the sources, but typically implies a lookup failure or default [S1, unofficial].
2. **Tint Initialization**: Initialize the tint for each face. If no specific tint is provided for a face, default to `Vector3.One` (white) [S1, unofficial].
3. **Instance Generation**: Pass the resolved texture IDs, tints, and shape reference to the `BlockRenderData` constructor.
4. **Face Tint Mixing**: The system supports mixing provided face tints with base block tints. This is performed via element-wise multiplication of the color vectors before instance generation. Alternatively, provided face tints can replace base block tints entirely [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class BlockRenderDataFactory:
    def __init__(self, texture_name_to_id_map):
        # texture_name_to_id_map is a dictionary mapping string names to int IDs
        self.texture_name_to_id_map = texture_name_to_id_map
        self.white_tint = Vector3(1.0, 1.0, 1.0)

    def create_block_render_data(self, 
                                 face_back_name, face_front_name, face_top_name, 
                                 face_bottom_name, face_left_name, face_right_name,
                                 tint_back=None, tint_front=None, tint_top=None, 
                                 tint_bottom=None, tint_left=None, tint_right=None,
                                 shape=None):
        """
        Creates a BlockRenderData instance.
        Tints default to white if not provided.
        """
        # Resolve texture IDs
        face_back_id = self.texture_name_to_id_map[face_back_name]
        face_front_id = self.texture_name_to_id_map[face_front_name]
        face_top_id = self.texture_name_to_id_map[face_top_name]
        face_bottom_id = self.texture_name_to_id_map[face_bottom_name]
        face_left_id = self.texture_name_to_id_map[face_left_name]
        face_right_id = self.texture_name_to_id_map[face_right_name]
        
        # Resolve tints, defaulting to white
        if tint_back is None: tint_back = self.white_tint
        if tint_front is None: tint_front = self.white_tint
        if tint_top is None: tint_top = self.white_tint
        if tint_bottom is None: tint_bottom = self.white_tint
        if tint_left is None: tint_left = self.white_tint
        if tint_right is None: tint_right = self.white_tint
        
        # Optional: Mix tints with base block tints (element-wise multiplication)
        # This step depends on external context not fully detailed in the source snippet
        # but implied by "Face Tint Mixing" alias.
        
        return BlockRenderData(
            face_back=face_back_id,
            face_front=face_front_id,
            face_top=face_top_id,
            face_bottom=face_bottom_id,
            face_left=face_left_id,
            face_right=face_right_id,
            tint_back=tint_back,
            tint_front=tint_front,
            tint_top=tint_top,
            tint_bottom=tint_bottom,
            tint_left=tint_left,
            tint_right=tint_right,
            shape=shape
        )

class BlockRenderData:
    def __init__(self, face_back, face_front, face_top, face_bottom, face_left, face_right,
                 tint_back, tint_front, tint_top, tint_bottom, tint_left, tint_right, shape):
        self.face_back = face_back
        self.face_front = face_front
        self.face_top = face_top
        self.face_bottom = face_bottom
        self.face_left = face_left
        self.face_right = face_right
        self.tint_back = tint_back
        self.tint_front = tint_front
        self.tint_top = tint_top
        self.tint_bottom = tint_bottom
        self.tint_left = tint_left
        self.tint_right = tint_right
        self.shape = shape
```

### Parameters
- **Default Tint**: `Vector3(1, 1, 1)` (White) [S1, unofficial].
- **Texture ID Source**: Derived from a `FrozenDictionary` mapping names to indices [S1, unofficial].

### Edge Cases and Pitfalls
- **Missing Texture Names**: The source uses dictionary lookup (`textureIDByName[...]`). If a texture name is not in the map, a `KeyError` or equivalent exception will occur. The sources do not specify a fallback behavior for missing names [S1, unofficial].
- **Tint Mixing**: The sources mention mixing tints via element-wise multiplication or replacement. The exact point in the pipeline where "base block tints" are defined and mixed is not detailed in the provided snippet, only that the mixing occurs before instance generation [S1, unofficial].

## Subtopics

### Chunk Mesh Assembly
Handles the bulk assembly of vertices and indices from `BlockRenderData` instances into mesh buffers. [Chunk Mesh Assembly](4100-chunk-mesh-assembly.md)

### Mesh Plugin Architecture
Defines how different mesh generation strategies, including block rendering, are integrated into the renderer. [Mesh Plugin Architecture](4921-mesh-plugin-architecture.md)

### Mesh Finalization
Processes the assembled mesh for rendering, including bounds calculation and material assignment. [Mesh Finalization](11478-mesh-finalization.md)

## Operating characteristics
- **Time Complexity**: $O(1)$ per block instance for data structure creation, assuming dictionary lookups are $O(1)$.
- **Memory Complexity**: $O(1)$ per block instance. The `BlockRenderData` struct is small and fixed-size.
- **Strengths**: Simple, explicit per-face control over textures and colors. Easy to implement.
- **Limitations**: High memory overhead if storing `BlockRenderData` for every voxel in a large world, as it stores 6 texture IDs and 6 tints per block. Not given in the sources: specific performance benchmarks or memory usage statistics.

## Background
This method originates from the `Cubic-Grid_Voxel_Renderer` project by rarefushion. It is part of a blocky meshing approach where voxels are rendered as individual quads or cubes with per-face material properties. The "Face Tint Mixing" alias suggests a specific feature for color modulation.

## Sources
[S1] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
