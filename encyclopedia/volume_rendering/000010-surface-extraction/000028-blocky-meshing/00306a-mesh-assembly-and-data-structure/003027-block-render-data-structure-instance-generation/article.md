# Block Render Data Structure & Instance Generation

Also called: BlockRenderData, Face Tint Mixing

## Overview
This method defines the data structure and generation process for individual block instances within a blocky meshing pipeline. It solves the problem of associating geometric primitives with specific visual properties (textures and color tints) per face, enabling efficient GPU instancing. It differs from sibling methods by focusing on the per-block metadata aggregation rather than the bulk vertex/index buffer assembly.

## How it works
**Inputs:**
*   A mapping of texture names to integer IDs (`textureIDByName`).
*   A set of face texture names (strings) for Back, Front, Top, Bottom, Left, Right.
*   A set of face tint colors (`Vector3`) for Back, Front, Top, Bottom, Left, Right.
*   A shape reference (`IShape`) defining the geometry.

**Outputs:**
*   A `BlockRenderData` struct instance containing the resolved texture IDs, tints, and shape.

**Data Structures:**
The core structure is `BlockRenderData`. It contains:
*   `faceBack`, `faceFront`, `faceTop`, `faceBottom`, `faceLeft`, `faceRight`: `int` fields storing texture IDs.
*   `tintBack`, `tintFront`, `tintTop`, `tintBottom`, `tintLeft`, `tintRight`: `Vector3` fields storing RGB tint values.
*   `shape`: `IShape` reference.

**Steps:**
1.  **Texture ID Lookup:** For each face direction (Back, Front, Top, Bottom, Left, Right), look up the texture name in `textureIDByName` to retrieve the corresponding integer ID. If a name is not found, the behavior is undefined in the sources [S1, unofficial].
2.  **Tint Handling:**
    *   If specific tints are provided for each face, use them directly.
    *   If a single tint is provided for all faces, apply it to all six faces.
    *   If no tint is provided, default to `Vector3(1, 1, 1)` (white) for all faces [S1, unofficial].
3.  **Instance Generation:** Pass the constructed `BlockRenderData` to the shape's `Instance` method. This method collects the tints for visible faces and generates the GPU instances.
4.  **Tint Mixing (Optional):** Before instance generation, face tints can be mixed with base block tints. The sources describe two approaches:
    *   Element-wise multiplication of provided face tints with base block tints.
    *   Replacement of base block tints entirely with provided face tints.
    The specific choice depends on the implementation context [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_block_render_data(
    texture_id_by_name: Dict[str, int],
    face_back_name: str,
    face_front_name: str,
    face_top_name: str,
    face_bottom_name: str,
    face_left_name: str,
    face_right_name: str,
    tint_back: Vector3,
    tint_front: Vector3,
    tint_top: Vector3,
    tint_bottom: Vector3,
    tint_left: Vector3,
    tint_right: Vector3,
    shape: IShape
) -> BlockRenderData:
    # Step 1: Lookup texture IDs
    face_back_id = texture_id_by_name[face_back_name]
    face_front_id = texture_id_by_name[face_front_name]
    face_top_id = texture_id_by_name[face_top_name]
    face_bottom_id = texture_id_by_name[face_bottom_name]
    face_left_id = texture_id_by_name[face_left_name]
    face_right_id = texture_id_by_name[face_right_name]

    # Step 2: Handle tints (default to white if not explicitly set in caller context)
    # Note: The source implies tints are passed in, but defaults are applied in helper methods.
    # Here we assume tints are provided. If a helper method is used, defaults are applied there.
    
    # Step 3: Construct the data structure
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

def create_with_names_only(
    texture_id_by_name: Dict[str, int],
    face_back_name: str,
    face_front_name: str,
    face_top_name: str,
    face_bottom_name: str,
    face_left_name: str,
    face_right_name: str,
    shape: IShape
) -> BlockRenderData:
    white = Vector3(1, 1, 1)
    return create_block_render_data(
        texture_id_by_name,
        face_back_name, face_front_name, face_top_name, face_bottom_name, face_left_name, face_right_name,
        white, white, white, white, white, white,
        shape
    )

def create_with_name_and_tint(
    texture_id_by_name: Dict[str, int],
    all_faces_name: str,
    all_tints: Vector3,
    shape: IShape
) -> BlockRenderData:
    return create_block_render_data(
        texture_id_by_name,
        all_faces_name, all_faces_name, all_faces_name, all_faces_name, all_faces_name, all_faces_name,
        all_tints, all_tints, all_tints, all_tints, all_tints, all_tints,
        shape
    )
```

**Parameters:**
*   `textureIDByName`: A dictionary mapping texture names to integer IDs.
*   `White`: Default tint `Vector3(1, 1, 1)`.

**Edge Cases:**
*   Missing texture names in `textureIDByName` will cause a lookup error.
*   The sources do not specify how to handle invalid or null shapes.

## Subtopics

## Operating characteristics
*   **Time Complexity:** O(1) per block, as it involves a fixed number of dictionary lookups and struct construction.
*   **Memory Complexity:** O(1) per block, storing 6 ints, 6 vectors, and 1 reference.
*   **Strengths:** Simple, efficient per-block metadata aggregation.
*   **Limitations:** Does not handle bulk buffer assembly; relies on downstream methods for GPU instance generation.

## Background
This method is part of the `Cubic-Grid_Voxel_Renderer` project by rarefushion. It provides the foundational data structure for defining block appearance in a voxel-based rendering system.

## Sources
[S1] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
