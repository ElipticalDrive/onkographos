# Voxel Backface Culling and Internal Face Masking

Also called: Internal Face Masking, Voxel Backface Culling

## Overview
Voxel Backface Culling and Internal Face Masking is a technique to reject invisible faces in voxel rendering by determining front/back orientation in projection space and masking faces shared between neighboring voxels [S1, unofficial]. It sets itself apart from sibling methods like [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) by operating directly on the geometry shader stage using projection-space dot products and bit-packed visibility codes, rather than pre-computing coverage masks or relying solely on neighbor occupancy checks during mesh generation.

## How it works
**Inputs and Outputs**
*   **Input:** Voxel corner positions in world space (`c_vCorner000` to `c_vCorner111`), view direction (`vViewDirection`), world-view matrix (`c_mWorldView`), projection matrix (`mProjection`), and a 32-bit voxel code (`vs_out_VoxelCode[0]`) encoding visibility masks and position [S1, unofficial].
*   **Output:** Vertex stream for visible front or back faces, with lighting computed per face [S1, unofficial].

**Data Structures**
*   **Voxel Code:** A 32-bit integer encoding invisible masking and position [S1, unofficial].
    *   Bits 0-5: Mask for each of the 6 sides (1 bit per side).
    *   Bits 6-31: Position information (3 * 8 bits for x, y, z coordinates).
    *   Bits 32-33: Unused (2 bits) [S1, unofficial].
    *   This structure allows addressing $256^3$ positions, necessitating a chunked approach for larger models [S1, unofficial].

**Steps**
1.  **Bounding Box Check:** Calculate the maximum length of the voxel corners. If the voxel is outside the viewing volume (defined by $|vPos.z| > w$, $|vPos.x| > w$, or $|vPos.y| > w$ where $w$ is the max corner length plus `vPos.w`), discard the voxel [S1, unofficial].
2.  **Transform Projection Direction:** Transform the direction in projection space to view space using the inverse diagonal of the projection matrix [S1, unofficial]:
    $$v_{view} = v_{proj} * (1/mProjection[0][0], 1/mProjection[1][1], 1/mProjection[2][2])$$
    [S1, unofficial]
3.  **Determine Face Orientation:** Calculate the dot product of the sum of two voxel corners (transformed by the inverse projection diagonal) and the view direction [S1, unofficial]:
    $$dot((c_{vCorner000}.xyz + c_{vCorner110}.xyz) * c_{vInverseProjection}.xyz, vViewDirection)$$
    [S1, unofficial]
4.  **Process Front Face:** If the dot product is greater than 0, process the front face [S1, unofficial].
    *   Check if the voxel code bit `0x10` is set in `vs_out_VoxelCode[0]` [S1, unofficial].
    *   If set, emit the front face vertices: `c_vCorner000`, `c_vCorner100`, `c_vCorner010`, `c_vCorner110` [S1, unofficial].
    *   Compute lighting with normal $(0,0,-1)$ transformed by the world-view matrix [S1, unofficial].
5.  **Process Back Face:** If the dot product is not greater than 0, process the back face [S1, unofficial].
    *   Check if the voxel code bit `0x20` is set in `vs_out_VoxelCode[0]` [S1, unofficial].
    *   If set, emit the back face vertices: `c_vCorner011`, `c_vCorner111`, `c_vCorner001`, `c_vCorner101` [S1, unofficial].
    *   Compute lighting with normal $(0,0,1)$ transformed by the world-view matrix [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_voxel(c_vCorner000, c_vCorner110, c_vCorner100, c_vCorner010, 
                  c_vCorner011, c_vCorner111, c_vCorner001, c_vCorner101,
                  vPos, vViewDirection, mProjection, c_mWorldView, 
                  vs_out_VoxelCode, shininess, specular, color, emissive):
    # Step 1: Bounding Box Check
    max_corner_length = max(length(c_vCorner000), length(c_vCorner001), 
                            length(c_vCorner010), length(c_vCorner110))
    w = vPos.w + max_corner_length
    if abs(vPos.z) > w or abs(vPos.x) > w or abs(vPos.y) > w:
        return # Discard voxel

    # Step 2: Transform Projection Direction to View Space
    # Assuming c_vInverseProjection is precomputed as (1/mProjection[0][0], 1/mProjection[1][1], 1/mProjection[2][2])
    v_view = vProj * c_vInverseProjection.xyz # vProj is implicit in the dot product calculation below

    # Step 3: Determine Face Orientation
    # Note: The source uses c_vCorner000.xyz + c_vCorner110.xyz transformed by inverse projection
    sum_corners = (c_vCorner000.xyz + c_vCorner110.xyz) * c_vInverseProjection.xyz
    dot_product = dot(sum_corners, vViewDirection)

    # Step 4: Process Front Face
    if dot_product > 0:
        # Check bit 0x10 (bit 4)
        if (vs_out_VoxelCode[0] & 0x10) != 0:
            # Emit front face vertices
            emit_vertex(c_vCorner000 + vPos)
            emit_vertex(c_vCorner100 + vPos)
            emit_vertex(c_vCorner010 + vPos)
            emit_vertex(c_vCorner110 + vPos)
            # Compute lighting with normal (0,0,-1) transformed by world-view matrix
            vNormal = normalize((vec4(0, 0, -1, 0) * c_mWorldView).xyz)
            gs_color = Lightning(vNormal, vViewDirection, shininess, specular, color, emissive)
            end_primitive()

    # Step 5: Process Back Face
    else:
        # Check bit 0x20 (bit 5)
        if (vs_out_VoxelCode[0] & 0x20) != 0:
            # Emit back face vertices
            emit_vertex(c_vCorner011 + vPos)
            emit_vertex(c_vCorner111 + vPos)
            emit_vertex(c_vCorner001 + vPos)
            emit_vertex(c_vCorner101 + vPos)
            # Compute lighting with normal (0,0,1) transformed by world-view matrix
            vNormal = normalize((vec4(0, 0, 1, 0) * c_mWorldView).xyz)
            gs_color = Lightning(vNormal, vViewDirection, shininess, specular, color, emissive)
            end_primitive()
```

**Parameters**
*   **Chunk Size:** $32^3$ voxels per chunk, used for efficient memory management, LOD, and large-scale culling [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Shared Faces:** Shared faces between two neighboring voxels are never visible. The masking bits (`0x10` and `0x20`) are used to reject these invisible internal faces [S1, unofficial].
*   **Projection Space Limitations:** Backface culling cannot be done directly in projection space; transformation to view space is required [S1, unofficial].

## Operating characteristics
*   **Performance:** Draws 32,368 voxels per millisecond on a Geforce 640M LE [S1, unofficial].
*   **Complexity:** Not given in the sources.
*   **Strengths:** Efficient rejection of internal faces and backfaces using hardware-accelerated geometry shaders and bit-packed codes [S1, unofficial].
*   **Limitations:** Geometry shaders can be slow if the amount of data increases significantly [S1, unofficial]. Requires a chunked approach for large models due to the 256^3 position addressing limit [S1, unofficial].
*   **When to use:** Suitable for voxel rendering where backfaces and internal faces need to be rejected early in the pipeline, especially on hardware with capable geometry shaders [S1, unofficial].
*   **When not to use:** Not suitable for very large models without chunking, or when geometry shader performance is a bottleneck [S1, unofficial].

## Background
This technique was developed by Johannes Jendersie and described in his graphic developer blog [S1, unofficial]. It was part of an effort to optimize voxel rendering, specifically for drawing beams and grids, by adding backface culling and internal face masking to the geometry shader stage [S1, unofficial]. The approach was tested on a Geforce 640M LE [S1, unofficial].

## Sources
[S1] Graphic | Developerblog – Johannes Jendersie, https://jojendersie.de/category/graphic/, fetched 2026-10-24 (unofficial)
