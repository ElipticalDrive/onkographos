# Reverse Index Map for Voxel Location
Also called: P->V Mapping
## Overview
The Reverse Index Map for Voxel Location is a technique used to determine the voxel location in volume coordinates ($V$) from the fragment location in the framebuffer plus the current slice z-offset, using a reverse index map ($P \rightarrow V$) [S1, unofficial].
## How it works
**Inputs:** Fragment location in the framebuffer, current slice z-offset [S1, unofficial].
**Outputs:** Voxel location in volume coordinates ($V$) [S1, unofficial].
**Data Structures:**
*   **Reverse Index Texture:** Maps packed coordinates ($P$) to volume coordinates ($V$) [S1, unofficial]. The texture stores the volume coordinate corresponding to each packed index [S1, unofficial].

**Steps:**
1.  Retrieve the fragment location in the framebuffer and the current slice z-offset [S1, unofficial].
2.  Use the reverse index map ($P \rightarrow V$) to determine the voxel location in volume coordinates [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_voxel_location(fragment_pos, slice_z_offset, reverse_index_texture):
    """
    Determines voxel location in volume coordinates from framebuffer fragment.
    
    Args:
        fragment_pos: Tuple (x, y) representing the fragment location in the framebuffer.
        slice_z_offset: Integer or float representing the current slice z-offset.
        reverse_index_texture: A lookup structure (texture) mapping packed coordinates P to volume coordinates V.
        
    Returns:
        V: The voxel location in volume coordinates.
    """
    # The source does not specify how fragment_pos and slice_z_offset are packed into P.
    # [G] Implementation detail: P must be constructed such that it uniquely identifies the voxel
    # corresponding to the fragment and slice. A common approach is P = (x, y, z_offset).
    packed_coord = (fragment_pos[0], fragment_pos[1], slice_z_offset)
    
    # Clamp packed coordinate to valid texture bounds to prevent out-of-bounds access.
    # [G] Standard guard for texture lookups.
    packed_coord = tuple(max(0, min(c, dim - 1)) for c, dim in zip(packed_coord, reverse_index_texture.dimensions))
    
    # Use reverse index map to determine voxel location.
    voxel_location = reverse_index_texture[packed_coord]
    
    return voxel_location
```

**Edge Cases and Pitfalls:**
*   **Precision:** The source does not specify precision issues. [G] Ensure that the packed coordinate $P$ does not suffer from integer overflow if coordinates are large.
*   **Boundaries:** If the fragment location is outside the valid framebuffer range or the slice offset is invalid, the lookup may fail. [G] Clamp inputs to valid ranges before packing.
*   **Degenerate Input:** If the volume is empty or the slice is out of bounds, the reverse index map may return undefined values. [G] Handle null or invalid volume coordinates.

## Subtopics

### Forward Index Map for Texture Indirection
A technique used during ray integration to map volume coordinates to packed texture coordinates via a forward mapping index texture ($V \rightarrow P$) [S1, unofficial]. It is used in the integration step where each sample requires a texture indirection from volume coordinates to packed texture coordinates [S1, unofficial]. The Forward Index Texture maps volume coordinates ($V$) to packed texture coordinates ($P$) [S1, unofficial]. It is chosen when ray integration requires accessing scalar data from a packed volume representation, complementing the Reverse Index Map which is used for initial voxel selection [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** Not given in the sources.
*   **Memory:** Not given in the sources.

**Measured Performance:**
*   The source mentions that rendering is interleaved with intensity texture updates, allowing user interaction, although all ray directions may require several frames to complete [S1, unofficial].
*   The technique uses adaptive sampling, which achieves a speedup of the order of the cube root of the data reduction ratio [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Enables interactive manipulation by quickly mapping screen coordinates to volume coordinates [S1, unofficial].
*   **Limitations:** Requires additional memory for the index textures [S1, unofficial]. The source does not specify performance bottlenecks beyond the multi-frame completion for all ray directions [S1, unofficial].

**When to Use:**
*   Use when direct manipulation segmentation requires fast mapping from framebuffer fragments to volume voxels [S1, unofficial].

## Background
The technique is described in the context of Local Ambient Occlusion in Direct Volume Rendering [S1, unofficial]. It is used in a rendering pass that is interleaved with intensity texture updates to allow user interaction [S1, unofficial]. The rendering uses raycasting with adaptive sampling [S1, unofficial].

## Sources
[S1] Local Ambient Occlusion in Direct Volume Rendering, https://www.researchgate.net/publication/44596498_Local_Ambient_Occlusion_in_Direct_Volume_Rendering, fetched 2026-10-28 (unofficial)
