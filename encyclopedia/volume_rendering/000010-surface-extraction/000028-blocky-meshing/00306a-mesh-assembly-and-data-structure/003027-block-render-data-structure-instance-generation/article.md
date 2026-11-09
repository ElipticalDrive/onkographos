# Block Render Data Structure & Instance Generation

Also called: BlockRenderData, Face Tint Mixing

## Overview
This method defines the data structure and generation logic for preparing voxel blocks for instance rendering. It maps block face textures and color tints to integer IDs and vector attributes, resolving these per-voxel properties before passing them to the rendering engine's instance API [S1, unofficial]. It differs from sibling methods by focusing on the per-instance attribute resolution (texture IDs and tints) rather than the global mesh buffer assembly or plugin architecture.

## How it works
**Inputs and Outputs:**
*   **Input:** A list of block definitions specifying face texture names (strings) and optional face-specific tints (vectors). A texture loader mapping texture names to integer IDs.
*   **Output:** A collection of `BlockRenderData` instances, each containing resolved texture IDs, tints, and a reference to the geometry shape.

**Data Structures:**
The core structure is `BlockRenderData`. It contains 13 fields:
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

**Steps:**
1.  **Texture ID Resolution:** For each of the six directions (Back, Front, Top, Bottom, Left, Right), fetch the texture ID. This is typically done by looking up the texture name in a dictionary (`textureIDByName`) or using a switch statement on the `Direction` enum if IDs are pre-mapped [S1, unofficial].
2.  **Tint Resolution:** Fetch the tint color for each face. If no specific tint is provided for a face, it defaults to white (`Vector3(1, 1, 1)`) [S1, unofficial].
3.  **Tint Mixing (Optional):** If external face tints are provided that override the base block tints, they are mixed with the base block tints. The mixing operation is element-wise multiplication: $T_{final} = T_{base} \times T_{override}$ [S1, unofficial]. Alternatively, the provided face tints may entirely replace the base block tints [S1, unofficial].
4.  **Instance Generation:** Collect the resolved `BlockRenderData` for all visible faces. Pass these instances to the shape's `Instance` method, which handles the GPU-side instance buffer population [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_block_render_data(face_names, face_tints, shape, texture_id_map):
    """
    Creates a BlockRenderData instance.
    
    Args:
        face_names: List of 6 strings [Back, Front, Top, Bottom, Left, Right]
        face_tints: List of 6 Vector3s [Back, Front, Top, Bottom, Left, Right]
        shape: IShape reference
        texture_id_map: Dictionary mapping texture name string to int ID
        
    Returns:
        BlockRenderData object
    """
    # Default tint is white
    WHITE = Vector3(1.0, 1.0, 1.0)
    
    # Resolve texture IDs
    face_back_id = texture_id_map[face_names[0]]
    face_front_id = texture_id_map[face_names[1]]
    face_top_id = texture_id_map[face_names[2]]
    face_bottom_id = texture_id_map[face_names[3]]
    face_left_id = texture_id_map[face_names[4]]
    face_right_id = texture_id_map[face_names[5]]
    
    # Resolve tints, defaulting to white if not provided or if list is shorter
    tint_back = face_tints[0] if len(face_tints) > 0 else WHITE
    tint_front = face_tints[1] if len(face_tints) > 1 else WHITE
    tint_top = face_tints[2] if len(face_tints) > 2 else WHITE
    tint_bottom = face_tints[3] if len(face_tints) > 3 else WHITE
    tint_left = face_tints[4] if len(face_tints) > 4 else WHITE
    tint_right = face_tints[5] if len(face_tints) > 5 else WHITE
    
    # Note: Mixing logic (element-wise multiplication) is applied if 
    # face_tints are overrides against a base block tint. 
    # If face_tints are the final values, they are used directly.
    
    return BlockRenderData(
        faceBack=face_back_id,
        faceFront=face_front_id,
        faceTop=face_top_id,
        faceBottom=face_bottom_id,
        faceLeft=face_left_id,
        faceRight=face_right_id,
        tintBack=tint_back,
        tintFront=tint_front,
        tintTop=tint_top,
        tintBottom=tint_bottom,
        tintLeft=tint_left,
        tintRight=tint_right,
        shape=shape
    )
```

**Parameters:**
*   `WHITE` tint: `Vector3(1, 1, 1)` [S1, unofficial].

**Edge Cases:**
*   **Missing Texture Names:** If a texture name is not in `textureIDByName`, the lookup will fail. The source implies a `FrozenDictionary` is used, suggesting all valid textures are pre-loaded [S1, unofficial].
*   **Missing Tints:** If fewer than 6 tints are provided, the remaining faces default to white [S1, unofficial].

## Subtopics

## Operating characteristics
*   **Time Complexity:** $O(1)$ per block instance generation, assuming dictionary lookups are $O(1)$.
*   **Memory Complexity:** $O(N)$ where $N$ is the number of visible faces, storing one `BlockRenderData` struct per instance.
*   **Strengths:** Simple, direct mapping from logical block definitions to GPU instance attributes. Supports per-face texture and tint variation.
*   **Limitations:** Requires all texture IDs to be known and mapped at creation time. Does not handle mesh buffer assembly itself.
*   **When to use:** When using instance rendering for voxel blocks and needing per-face texture/tint control.
*   **When not to use:** When using non-instance rendering or when texture IDs are dynamic and not pre-mapped.

## Background
This method originates from the `Cubic-Grid_Voxel_Renderer` project by rarefushion [S1, unofficial]. It is part of a voxel rendering pipeline that separates block data definition from mesh assembly and rendering.

## Sources
[S1] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
