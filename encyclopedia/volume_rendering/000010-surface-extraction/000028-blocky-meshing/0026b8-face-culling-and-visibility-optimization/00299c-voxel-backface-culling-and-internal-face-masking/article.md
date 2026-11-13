# Voxel Backface Culling and Internal Face Masking

Also called: Internal Face Masking, Voxel Backface Culling

## Overview
Voxel Backface Culling and Internal Face Masking is a visibility optimization technique for voxel-based surface extraction that rejects invisible faces by determining front/back orientation in projection space and masking faces shared between neighboring voxels [S1, unofficial]. It sets itself apart from sibling methods like [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) by performing backface culling directly in the geometry shader using projection-space heuristics, rather than relying solely on pre-computed neighbor occupancy checks or coverage masks.

## How it works
**Inputs and Outputs**
*   **Input:** A voxel chunk (typically $32^3$ voxels) [S1, unofficial]. For each voxel, the method requires:
    *   `c_vCorner000` to `c_vCorner111`: The 8 world-space corner positions of the voxel [S1, unofficial].
    *   `vPos`: The world-space translation offset for the current chunk [S1, unofficial].
    *   `vViewDirection`: The view direction vector in world space [S1, unofficial].
    *   `c_mWorldView`: The world-view matrix [S1, unofficial].
    *   `c_vInverseProjection`: The inverse projection diagonal vector, calculated as $(1/mProjection[0][0], 1/mProjection[1][1], 1/mProjection[2][2])$ [S1, unofficial].
    *   `vs_out_VoxelCode[0]`: A 32-bit integer encoding visibility masks and position [S1, unofficial].
*   **Output:** A stream of vertices forming triangles for only the visible, non-internal faces of the voxel [S1, unofficial].

**Data Structures**
*   **Voxel Code:** A 32-bit integer where:
    *   Bits 0-5 (6 bits): Mask for each side's visibility [S1, unofficial].
    *   Bits 6-30 (25 bits): Position information (3 * 8 bits for x, y, z) [S1, unofficial].
    *   Bits 31-32 (2 bits): Unused [S1, unofficial].
    *   This structure allows addressing $256^3$ positions [S1, unofficial].

**Steps**
1.  **Viewing Volume Check:** Determine if the voxel is within the viewing volume. Calculate $w$ as the maximum length of the voxel's corners plus the position's w-component:
    $$ w = vPos.w + \max(\max(\|c\_vCorner000\|, \|c\_vCorner001\|), \max(\|c\_vCorner010\|, \|c\_vCorner011\|)) $$
    If $|vPos.z| > w$ or $|vPos.x| > w$ or $|vPos.y| > w$, discard the voxel [S1, unofficial].

2.  **Backface Determination:** Calculate the dot product to decide between front and back face processing. Transform the projection space direction to view space using the inverse projection diagonal:
    $$ v_{view} = v_{proj} * (1/mProjection[0][0], 1/mProjection[1][1], 1/mProjection[2][2]) $$
    Compute the decision metric:
    $$ \text{metric} = \text{dot}((c\_vCorner000.xyz + c\_vCorner110.xyz) * c\_vInverseProjection.xyz, vViewDirection) $$
    If $\text{metric} > 0$, process the front face; otherwise, process the back face [S1, unofficial].

3.  **Internal Face Masking and Emission:**
    *   **Front Face:** If the metric $> 0$, check if bit `0x10` is set in `vs_out_VoxelCode[0]`. If set, emit the front face vertices: $c\_vCorner000, c\_vCorner100, c\_vCorner010, c\_vCorner110$ [S1, unofficial]. Compute lighting using the normal $(0,0,-1)$ transformed by the world-view matrix [S1, unofficial].
    *   **Back Face:** If the metric $\le 0$, check if bit `0x20` is set in `vs_out_VoxelCode[0]`. If set, emit the back face vertices: $c\_vCorner011, c\_vCorner111, c\_vCorner001, c\_vCorner101$ [S1, unofficial]. Compute lighting using the normal $(0,0,1)$ transformed by the world-view matrix [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_voxel(c_vCorner000, c_vCorner001, c_vCorner010, c_vCorner011,
                  c_vCorner100, c_vCorner101, c_vCorner110, c_vCorner111,
                  vPos, vViewDirection, c_mWorldView, c_vInverseProjection,
                  vs_out_VoxelCode):
    # Step 1: Viewing Volume Check
    w = vPos.w + max(
        max(length(c_vCorner000), length(c_vCorner001)),
        max(length(c_vCorner010), length(c_vCorner011))
    )
    if abs(vPos.z) > w or abs(vPos.x) > w or abs(vPos.y) > w:
        return [] # Discard voxel

    # Step 2: Backface Determination
    # Note: c_vInverseProjection is assumed to be (1/mProj[0][0], 1/mProj[1][1], 1/mProj[2][2])
    sum_corners = c_vCorner000.xyz + c_vCorner110.xyz
    scaled_corners = sum_corners * c_vInverseProjection.xyz
    metric = dot(scaled_corners, vViewDirection)

    vertices = []

    # Step 3: Internal Face Masking and Emission
    if metric > 0:
        # Front Face
        if (vs_out_VoxelCode[0] & 0x10) != 0:
            # Emit front face vertices
            vertices.append(c_vCorner000 + vPos)
            vertices.append(c_vCorner100 + vPos)
            vertices.append(c_vCorner010 + vPos)
            vertices.append(c_vCorner110 + vPos)
            # Lighting calculation with normal (0,0,-1) transformed by world-view
            # vNormal = normalize((vec4(0,0,-1,0) * c_mWorldView).xyz)
            # gs_color = Lightning(vNormal, vViewDirection, ...)
    else:
        # Back Face
        if (vs_out_VoxelCode[0] & 0x20) != 0:
            # Emit back face vertices
            vertices.append(c_vCorner011 + vPos)
            vertices.append(c_vCorner111 + vPos)
            vertices.append(c_vCorner001 + vPos)
            vertices.append(c_vCorner101 + vPos)
            # Lighting calculation with normal (0,0,1) transformed by world-view
            # vNormal = normalize((vec4(0,0,1,0) * c_mWorldView).xyz)
            # gs_color = Lightning(vNormal, vViewDirection, ...)

    return vertices
```

**Parameters**
*   **Chunk Size:** $32^3$ voxels per chunk [S1, unofficial]. This size is chosen for efficient memory management, LOD, and large-scale culling algorithms [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Shared Faces:** Shared faces between two neighboring voxels are never visible. The masking bits (`0x10` and `0x20`) are used to reject these invisible internal faces [S1, unofficial].
*   **Projection Space Limitations:** Backface culling cannot be done directly in projection space without transformation. The inverse projection diagonal trick is required to approximate view-space orientation [S1, unofficial].

## Operating characteristics
*   **Performance:** On a Geforce 640M LE, the method draws 32,368 voxels per millisecond [S1, unofficial].
*   **Complexity:** The method operates per-voxel in the geometry shader. The cost is dominated by the vertex emission and lighting calculation for visible faces.
*   **Strengths:** Reduces triangle count by removing internal faces and backfaces. The 32-bit integer encoding allows efficient data transfer to the GPU [S1, unofficial].
*   **Limitations:** Geometry shaders can be slow if the amount of data increases significantly [S1, unofficial]. The backface culling heuristic is an approximation based on the inverse projection diagonal, which may not be perfectly accurate for all projection matrices or extreme viewing angles.
*   **When to use:** Suitable for real-time voxel rendering where performance is critical and the voxel data is chunked.
*   **When not to use:** When exact backface culling is required for all projection types, or when the overhead of geometry shaders is prohibitive compared to pre-computed mesh approaches.

## Background
This technique was developed by Johannes Jendersie and published on his developer blog [S1, unofficial]. It was part of an effort to optimize voxel rendering, specifically for drawing beams and grids, by reducing the number of triangles processed by the GPU. The approach combines backface culling with internal face masking to improve performance on hardware like the Geforce 640M LE [S1, unofficial].

## Sources
[S1] Graphic | Developerblog – Johannes Jendersie, https://jojendersie.de/category/graphic/, fetched 2026-10-24 (unofficial)
