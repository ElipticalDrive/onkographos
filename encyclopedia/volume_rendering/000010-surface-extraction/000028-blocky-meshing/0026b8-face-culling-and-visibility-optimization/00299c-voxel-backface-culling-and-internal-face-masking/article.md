# Voxel Backface Culling and Internal Face Masking

Also called: Internal Face Masking, Voxel Backface Culling

## Overview
Voxel Backface Culling and Internal Face Masking is a geometry optimization technique for blocky voxel meshing that rejects invisible faces by determining front/back orientation in projection space and masking faces shared between neighboring voxels [S1, unofficial]. It sets itself apart from sibling methods like [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) by performing visibility checks directly within the geometry shader using projection-space dot products and bit-packed voxel codes, rather than relying on pre-computed coverage masks or complex connected component analysis [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input**: A voxel chunk (typically $32^3$ voxels) represented as a grid of occupied cells. Each voxel carries a 32-bit integer code encoding its position and face visibility masks.
*   **Output**: A triangle mesh containing only visible front and back faces, with internal and back-facing triangles discarded.

**Data Structures**
The core data structure is a 32-bit integer (`vs_out_VoxelCode[0]`) that encodes both the voxel's position and the visibility state of its faces [S1, unofficial].
*   **Layout**:
    *   Bits 0-5 (6 bits): Mask for each of the 6 sides of the voxel.
    *   Bits 6-30 (24 bits): Position coordinates (3 * 8 bits for x, y, z).
    *   Bits 31-32 (2 bits): Unused.
*   **Position Encoding**: The 24-bit position field allows addressing a $256^3$ volume space [S1, unofficial]. Larger models require a chunked approach, with a standard chunk size of $32^3$ voxels to facilitate efficient memory management and LOD [S1, unofficial].

**Algorithm Steps**
The method is implemented in a geometry shader, processing each voxel individually.

1.  **Bounding Box Check**: Calculate the world-space bounding box of the voxel. If the voxel lies outside the viewing volume (frustum culling), discard it [S1, unofficial].
    *   Let $c_{vCorner000}$ to $c_{vCorner111}$ be the 8 corners of the voxel in world space.
    *   Let $vPos$ be the position offset.
    *   Compute $w = vPos.w + \max(\max(\|c_{vCorner000}\|, \|c_{vCorner001}\|), \max(\|c_{vCorner010}\|, \|c_{vCorner111}\|))$.
    *   If $|vPos.z| > w$ or $|vPos.x| > w$ or $|vPos.y| > w$, return [S1, unofficial].

2.  **Projection Space Transformation**: Transform the view direction from projection space to view space to enable backface culling.
    *   Let $M_{proj}$ be the projection matrix.
    *   Compute the inverse projection diagonal vector: $v_{invProj} = (1/M_{proj}[0][0], 1/M_{proj}[1][1], 1/M_{proj}[2][2])$.
    *   The view direction in projection space $v_{proj}$ is transformed to view space $v_{view}$ via: $v_{view} = v_{proj} * v_{invProj}$ [S1, unofficial].

3.  **Front/Back Face Determination**: Determine if the voxel's primary axis-aligned faces are front-facing or back-facing relative to the camera.
    *   Compute the dot product: $D = \text{dot}((c_{vCorner000}.xyz + c_{vCorner110}.xyz) * v_{invProj}.xyz, v_{view})$.
    *   If $D > 0$, the front face is visible; otherwise, the back face is visible [S1, unofficial].

4.  **Internal Face Masking**:
    *   **Front Face**: If the front face is determined to be visible, check if the voxel code bit `0x10` is set in `vs_out_VoxelCode[0]`.
        *   If set, emit the front face vertices: $c_{vCorner000}, c_{vCorner100}, c_{vCorner010}, c_{vCorner110}$.
        *   Compute lighting using the normal $\vec{n} = (0, 0, -1)$ transformed by the world-view matrix [S1, unofficial].
    *   **Back Face**: If the back face is determined to be visible, check if the voxel code bit `0x20` is set in `vs_out_VoxelCode[0]`.
        *   If set, emit the back face vertices: $c_{vCorner011}, c_{vCorner111}, c_{vCorner001}, c_{vCorner101}$.
        *   Compute lighting using the normal $\vec{n} = (0, 0, 1)$ transformed by the world-view matrix [S1, unofficial].

5.  **Vertex Emission**: For each emitted face, set `gl_Position` for each corner by adding the position offset $vPos$ to the corner coordinates, then call `EmitVertex()` and `EndPrimitive()` [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_voxel_geometry_shader(voxel_corners, voxel_code, view_direction_proj, projection_matrix, world_view_matrix, position_offset):
    """
    Processes a single voxel in the geometry shader.
    voxel_corners: dict with keys '000' to '111', each a vec3 world position.
    voxel_code: int, 32-bit integer encoding position and face masks.
    view_direction_proj: vec3, view direction in projection space.
    projection_matrix: mat4, the current projection matrix.
    world_view_matrix: mat4, the world-view matrix.
    position_offset: vec4, position offset for the voxel.
    """
    # Step 1: Frustum Culling
    corners = [voxel_corners['000'], voxel_corners['001'], voxel_corners['010'], voxel_corners['111']]
    max_len = max([length(c) for c in corners])
    w = position_offset.w + max_len
    
    if abs(position_offset.z) > w or abs(position_offset.x) > w or abs(position_offset.y) > w:
        return # Discard voxel

    # Step 2: Transform View Direction
    inv_proj_diag = vec3(
        1.0 / projection_matrix[0][0],
        1.0 / projection_matrix[1][1],
        1.0 / projection_matrix[2][2]
    )
    view_direction_view = view_direction_proj * inv_proj_diag

    # Step 3: Determine Front/Back
    # Sum of two opposite corners (e.g., bottom-left-front and top-right-back)
    corner_sum = voxel_corners['000'].xyz + voxel_corners['110'].xyz
    dot_product = dot(corner_sum * inv_proj_diag.xyz, view_direction_view)

    # Step 4 & 5: Emit Visible Faces
    if dot_product > 0:
        # Front Face
        if (voxel_code & 0x10) != 0:
            # Normal (0,0,-1) transformed by world-view matrix
            normal = normalize((vec4(0, 0, -1, 0) * world_view_matrix).xyz)
            # Emit vertices for front face
            emit_vertex(voxel_corners['000'] + position_offset)
            emit_vertex(voxel_corners['100'] + position_offset)
            emit_vertex(voxel_corners['010'] + position_offset)
            emit_vertex(voxel_corners['110'] + position_offset)
            end_primitive()
    else:
        # Back Face
        if (voxel_code & 0x20) != 0:
            # Normal (0,0,1) transformed by world-view matrix
            normal = normalize((vec4(0, 0, 1, 0) * world_view_matrix).xyz)
            # Emit vertices for back face
            emit_vertex(voxel_corners['011'] + position_offset)
            emit_vertex(voxel_corners['111'] + position_offset)
            emit_vertex(voxel_corners['001'] + position_offset)
            emit_vertex(voxel_corners['101'] + position_offset)
            end_primitive()
```

**Parameters**
*   **Chunk Size**: $32^3$ voxels per chunk [S1, unofficial].
*   **Voxel Code Bits**: `0x10` for front face visibility, `0x20` for back face visibility [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Shared Faces**: Faces shared between two neighboring voxels are never visible. The masking bits (`0x10` and `0x20`) are specifically used to reject these invisible internal faces [S1, unofficial].
*   **Projection Space Limitations**: Backface culling cannot be done directly in projection space without transformation; the inverse projection diagonal multiplication is required [S1, unofficial].

## Operating characteristics
*   **Performance**: On a GeForce 640M LE, the method draws 32,368 voxels per millisecond [S1, unofficial].
*   **Complexity**: The algorithm operates in $O(N)$ time relative to the number of voxels in the chunk, with constant-time bitwise checks for face visibility.
*   **Strengths**: Efficient rejection of internal faces and backfaces reduces the number of triangles sent to the rasterizer, lowering GPU load. The bit-packed encoding minimizes memory bandwidth for voxel data.
*   **Limitations**: Geometry shaders can be slow if the amount of data increases significantly. The method relies on a specific projection matrix structure for the inverse diagonal trick. It is limited to axis-aligned faces and does not handle complex transparent materials directly without additional logic (e.g., opaque separation).
*   **When to Use**: Suitable for large-scale voxel worlds where internal faces constitute a significant portion of the geometry and performance is critical.
*   **When Not to Use**: Not suitable for non-axis-aligned voxel representations or when complex transparency effects are required without additional passes.

## Background
The method was developed by Johannes Jendersie and published on his developer blog in 2013 [S1, unofficial]. It was part of a series of optimizations for a voxel engine, aiming to improve rendering performance by reducing the number of draw calls and triangles. The technique builds on the principles of face culling and visibility optimization, specifically targeting the inefficiencies of rendering all voxel faces regardless of visibility.

## Sources
[S1] Graphic | Developerblog – Johannes Jendersie, https://jojendersie.de/category/graphic/, fetched 2026-10-24 (unofficial)
