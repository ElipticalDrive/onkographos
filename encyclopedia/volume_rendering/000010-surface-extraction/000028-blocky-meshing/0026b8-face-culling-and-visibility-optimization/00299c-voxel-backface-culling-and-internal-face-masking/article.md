# Voxel Backface Culling and Internal Face Masking

Also called: Internal Face Masking, Voxel Backface Culling

## Overview
Voxel Backface Culling and Internal Face Masking is a visibility optimization technique for voxel-based rendering that rejects invisible faces by determining front/back orientation in projection space and masking faces shared between neighboring voxels [S1, unofficial]. It differs from standard geometric backface culling by operating on voxel codes to simultaneously handle internal face rejection (faces between two solid voxels) and view-dependent backface culling, allowing for efficient geometry shader-based rendering [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel chunk (typically $32^3$ voxels), a world-view matrix, a projection matrix, and a view direction vector. Each voxel is represented by a 32-bit integer encoding its position and face visibility masks [S1, unofficial].
*   **Output:** A stream of triangles (quads) for visible faces, processed by a geometry shader [S1, unofficial].

**Data Structures:**
The core data structure is a 32-bit integer per voxel, encoding invisible masking and position [S1, unofficial]:
*   Bits 0-5: Mask for each of the 6 sides (±X, ±Y, ±Z).
*   Bits 8-23: 3 * 8 bits for the voxel position (allowing addressing of $256^3$ positions).
*   Bits 6-7, 24-31: Unused (2 bits currently unused per source, though 32-bit total implies specific packing) [S1, unofficial].

**Steps:**
1.  **Transform Projection Direction:** Transform the direction in projection space to view space using the inverse diagonal of the projection matrix [S1, unofficial].
    $$ v_{view} = v_{proj} * (1/mProjection[0][0], 1/mProjection[1][1], 1/mProjection[2][2]) $$
    Where $v_{proj}$ is the direction in projection space and $mProjection$ is the projection matrix [S1, unofficial].

2.  **Determine Face Orientation:** Calculate the dot product of the sum of two voxel corners transformed by the inverse projection diagonal and the view direction [S1, unofficial].
    $$ \text{dot}((c_{vCorner000}.xyz + c_{vCorner110}.xyz) * c_{vInverseProjection}.xyz, vViewDirection) $$
    *   If the result is $> 0$, process the front face [S1, unofficial].
    *   Otherwise, process the back face [S1, unofficial].

3.  **Check Internal Masking:**
    *   **Front Face:** Check if the voxel code bit `0x10` is set in `vs_out_VoxelCode[0]` [S1, unofficial].
        *   If set, emit the front face vertices: $(c_{vCorner000}, c_{vCorner100}, c_{vCorner010}, c_{vCorner110})$ [S1, unofficial].
        *   Compute lighting with normal $(0, 0, -1)$ transformed by the world-view matrix [S1, unofficial].
    *   **Back Face:** Check if the voxel code bit `0x20` is set in `vs_out_VoxelCode[0]` [S1, unofficial].
        *   If set, emit the back face vertices: $(c_{vCorner011}, c_{vCorner111}, c_{vCorner001}, c_{vCorner101})$ [S1, unofficial].
        *   Compute lighting with normal $(0, 0, 1)$ transformed by the world-view matrix [S1, unofficial].

4.  **Cull Internal Faces:** Shared faces between two neighboring voxels are never visible. The masking bits (`0x10` and `0x20`) are used to reject these invisible internal faces [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_voxel(voxel_code, voxel_pos, world_view_matrix, projection_matrix, view_direction):
    # Extract position from voxel_code (bits 8-23)
    # Extract mask from voxel_code (bits 0-5)
    
    # Define corner offsets for a unit voxel at voxel_pos
    c_vCorner000 = voxel_pos + vec3(0, 0, 0)
    c_vCorner100 = voxel_pos + vec3(1, 0, 0)
    c_vCorner010 = voxel_pos + vec3(0, 1, 0)
    c_vCorner001 = voxel_pos + vec3(0, 0, 1)
    c_vCorner110 = voxel_pos + vec3(1, 1, 0)
    c_vCorner101 = voxel_pos + vec3(1, 0, 1)
    c_vCorner011 = voxel_pos + vec3(0, 1, 1)
    c_vCorner111 = voxel_pos + vec3(1, 1, 1)
    
    # Calculate inverse projection diagonal
    inv_proj_diag = vec3(
        1.0 / projection_matrix[0][0],
        1.0 / projection_matrix[1][1],
        1.0 / projection_matrix[2][2]
    )
    
    # Transform sum of corners to view space direction
    corner_sum = (c_vCorner000.xyz + c_vCorner110.xyz) * inv_proj_diag
    dot_product = dot(corner_sum, view_direction)
    
    if dot_product > 0:
        # Front face check
        if (voxel_code & 0x10) != 0:
            # Emit front face vertices
            emit_vertex(c_vCorner000)
            emit_vertex(c_vCorner100)
            emit_vertex(c_vCorner010)
            emit_vertex(c_vCorner110)
            # Compute lighting with normal (0,0,-1) transformed by world-view
            normal = normalize((vec4(0, 0, -1, 0) * world_view_matrix).xyz)
            color = compute_lighting(normal, view_direction)
    else:
        # Back face check
        if (voxel_code & 0x20) != 0:
            # Emit back face vertices
            emit_vertex(c_vCorner011)
            emit_vertex(c_vCorner111)
            emit_vertex(c_vCorner001)
            emit_vertex(c_vCorner101)
            # Compute lighting with normal (0,0,1) transformed by world-view
            normal = normalize((vec4(0, 0, 1, 0) * world_view_matrix).xyz)
            color = compute_lighting(normal, view_direction)
```

**Parameters:**
*   Chunk size: $32^3$ voxels per chunk [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Shared Faces:** Shared faces between two neighboring voxels are never visible. The masking bits must be correctly set during voxel generation to avoid rendering internal geometry [S1, unofficial].
*   **Projection Space Limitations:** Backface culling cannot be done directly in projection space without transformation to view space using the inverse projection diagonal [S1, unofficial].
*   **Performance:** Geometry shaders can be slow if the amount of data increases. Efficient memory management and chunking are necessary for large-scale culling [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Draws 32,368 voxels per millisecond on a Geforce 640M LE [S1, unofficial].
*   **Strengths:** Efficient rejection of internal faces and view-dependent backfaces in a single pass using geometry shaders [S1, unofficial].
*   **Limitations:** Geometry shader performance can degrade with large data amounts; requires careful chunking for large models [S1, unofficial].
*   **When to Use:** Suitable for voxel-based rendering where internal face visibility is a concern and geometry shaders are available [S1, unofficial].
*   **When Not to Use:** Not suitable for very large models without chunking or if geometry shader performance is a bottleneck [S1, unofficial].

## Background
This technique was developed by Johannes Jendersie and published on his developer blog [S1, unofficial]. It is part of a series of optimizations for voxel rendering, including beam drawing and grid rendering [S1, unofficial]. The approach leverages the properties of voxel grids to simplify visibility testing [S1, unofficial].

## Sources
[S1] Graphic | Developerblog – Johannes Jendersie, https://jojendersie.de/category/graphic/, fetched 2026-10-24 (unofficial)
