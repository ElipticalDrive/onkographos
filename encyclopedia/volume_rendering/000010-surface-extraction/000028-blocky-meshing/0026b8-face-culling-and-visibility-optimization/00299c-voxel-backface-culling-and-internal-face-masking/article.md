# Voxel Backface Culling and Internal Face Masking

Also called: Internal Face Masking, Voxel Backface Culling

## Overview
This method optimizes voxel mesh generation by rejecting invisible faces in projection space and masking faces shared between neighboring voxels. It addresses the performance bottleneck of rendering internal geometry by combining view-dependent backface culling with topology-based internal face rejection. It differs from standard neighbor-check culling [Face Culling](8735-face-culling.md) by explicitly handling projection-space orientation to reject back-facing surfaces, while retaining the internal masking logic to skip shared faces.

## How it works
**Inputs:**
*   Voxel position $P$ and corner coordinates $c\_vCorner_{ijk}$ (where $i,j,k \in \{0,1\}$).
*   View direction $v_{ViewDirection}$ in view space.
*   Projection matrix $m_{Projection}$.
*   World-view matrix $c\_mWorldView$.
*   Voxel code $vs\_out\_VoxelCode[0]$ (32-bit integer).

**Outputs:**
*   Vertex stream for visible front or back faces.

**Data Structures:**
*   **Voxel Code:** A 32-bit integer encoding visibility masks and position.
    *   Bits 0-5: Mask for each of the 6 sides (2 bits per side? The source specifies "6 bits for the mask of each side" but uses `0x10` and `0x20` which are single bits. The source text says "6 bits for the mask of each side" but the code implies specific bits control specific faces. We follow the code: Bit 4 (`0x10`) for front, Bit 5 (`0x20`) for back).
    *   Bits 6-31: Position information (3*8 bits for position, 2 unused).
    *   *Note:* The source states "6 bits for the mask of each side" but the implementation uses specific bits `0x10` and `0x20`. This implies a packed structure where specific bits correspond to specific faces.

**Steps:**
1.  **Frustum Check:** Reject voxels outside the viewing volume.
    *   Calculate $w = vPos.w + \max(\max(\|c\_vCorner_{000}\|, \|c\_vCorner_{001}\|), \max(\|c\_vCorner_{010}\|, \|c\_vCorner_{011}\|))$.
    *   If $|vPos.z| > w$ or $|vPos.x| > w$ or $|vPos.y| > w$, return. [S1, unofficial]
2.  **Projection Space Transformation:** Transform the projection space direction to view space.
    *   Compute inverse projection diagonal: $v_{InverseProjection} = (1/m_{Projection}[0][0], 1/m_{Projection}[1][1], 1/m_{Projection}[2][2])$. [S1, unofficial]
    *   The direction in projection space is transformed to view space using: $v_{view} = v_{proj} * v_{InverseProjection}$. [S1, unofficial]
3.  **Backface Culling Decision:** Determine if the front or back face is visible.
    *   Calculate dot product: $D = \text{dot}((c\_vCorner_{000}.xyz + c\_vCorner_{110}.xyz) * v_{InverseProjection}.xyz, v_{ViewDirection})$. [S1, unofficial]
    *   If $D > 0$, process the **front face**.
    *   If $D \le 0$, process the **back face**.
4.  **Internal Face Masking:** Check if the face is internal (shared with a neighbor) using the voxel code.
    *   **Front Face:** If processing front face, check if bit `0x10` is set in $vs\_out\_VoxelCode[0]$. If set, the face is visible (not internal). If not set, cull. [S1, unofficial]
    *   **Back Face:** If processing back face, check if bit `0x20` is set in $vs\_out\_VoxelCode[0]$. If set, the face is visible. If not set, cull. [S1, unofficial]
5.  **Vertex Emission:** If the face is visible, emit vertices and compute lighting.
    *   **Front Face Vertices:** $c\_vCorner_{000}, c\_vCorner_{100}, c\_vCorner_{010}, c\_vCorner_{110}$.
    *   **Front Face Normal:** $(0, 0, -1)$ transformed by world-view matrix: $v_{Normal} = \text{normalize}((vec4(0,0,-1,0) * c\_mWorldView).xyz)$. [S1, unofficial]
    *   **Back Face Vertices:** $c\_vCorner_{011}, c\_vCorner_{111}, c\_vCorner_{001}, c\_vCorner_{101}$.
    *   **Back Face Normal:** $(0, 0, 1)$ transformed by world-view matrix: $v_{Normal} = \text{normalize}((vec4(0,0,1,0) * c\_mWorldView).xyz)$. [S1, unofficial]
    *   Compute lighting using $v_{Normal}$, $v_{ViewDirection}$, and material properties. [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_voxel_face(c_vCorner000, c_vCorner110, c_vCorner100, c_vCorner010, 
                       c_vCorner011, c_vCorner111, c_vCorner001, c_vCorner101, 
                       vPos, vViewDirection, mProjection, c_mWorldView, vs_out_VoxelCode):
    # 1. Frustum Check
    w = vPos.w + max(max(length(c_vCorner000), length(c_vCorner001)), 
                     max(length(c_vCorner010), length(c_vCorner011)))
    if abs(vPos.z) > w or abs(vPos.x) > w or abs(vPos.y) > w:
        return

    # 2. Projection Space Transformation
    # Inverse projection diagonal
    inv_proj_x = 1.0 / mProjection[0][0]
    inv_proj_y = 1.0 / mProjection[1][1]
    inv_proj_z = 1.0 / mProjection[2][2]
    c_vInverseProjection = (inv_proj_x, inv_proj_y, inv_proj_z)

    # 3. Backface Culling Decision
    # Sum of corners for direction calculation
    corner_sum = (c_vCorner000.x + c_vCorner110.x, 
                  c_vCorner000.y + c_vCorner110.y, 
                  c_vCorner000.z + c_vCorner110.z)
    
    # Transform to view space direction
    transformed_dir = (corner_sum[0] * c_vInverseProjection[0],
                       corner_sum[1] * c_vInverseProjection[1],
                       corner_sum[2] * c_vInverseProjection[2])
    
    dot_product = dot(transformed_dir, vViewDirection)

    # 4. Internal Face Masking and Emission
    if dot_product > 0:
        # Front Face
        if (vs_out_VoxelCode[0] & 0x10) != 0:
            # Emit Front Face Vertices
            emit_vertex(c_vCorner000 + vPos)
            emit_vertex(c_vCorner100 + vPos)
            emit_vertex(c_vCorner010 + vPos)
            emit_vertex(c_vCorner110 + vPos)
            # Compute Normal
            normal_vec = vec4(0, 0, -1, 0)
            vNormal = normalize((normal_vec * c_mWorldView).xyz)
            compute_lighting(vNormal, vViewDirection)
    else:
        # Back Face
        if (vs_out_VoxelCode[0] & 0x20) != 0:
            # Emit Back Face Vertices
            emit_vertex(c_vCorner011 + vPos)
            emit_vertex(c_vCorner111 + vPos)
            emit_vertex(c_vCorner001 + vPos)
            emit_vertex(c_vCorner101 + vPos)
            # Compute Normal
            normal_vec = vec4(0, 0, 1, 0)
            vNormal = normalize((normal_vec * c_mWorldView).xyz)
            compute_lighting(vNormal, vViewDirection)
```

**Parameters:**
*   **Chunk Size:** $32^3$ voxels per chunk. This size is chosen for efficient memory management, LOD, and large-scale culling algorithms. [S1, unofficial]

**Edge Cases and Pitfalls:**
*   **Shared Faces:** Shared faces between two neighboring voxels are never visible. The masking bits (`0x10` and `0x20`) are used to reject these invisible internal faces. [S1, unofficial]
*   **Projection Matrix Assumption:** The transformation assumes a specific projection matrix structure where the inverse diagonal can be used directly. [S1, unofficial]
*   **Geometry Shader Performance:** Geometry shaders can be slow with large data amounts. The encoding of masking and position into a single 32-bit integer helps mitigate this. [S1, unofficial]

## Operating characteristics
*   **Performance:** On a Geforce 640M LE, the method draws 32,368 voxels per millisecond. [S1, unofficial]
*   **Complexity:** The method adds a dot product calculation and bitwise checks per voxel face. The frustum check is a simple bounding box test.
*   **Memory:** The voxel code is packed into a 32-bit integer, allowing 256^3 positions to be addressed. [S1, unofficial]
*   **Strengths:** Reduces overdraw by rejecting back-facing and internal faces. Efficient memory usage via packed voxel codes.
*   **Limitations:** Relies on geometry shaders, which may have performance implications on some hardware. The projection space transformation assumes a specific projection matrix format.

## Background
This technique was developed by Johannes Jendersie for the game "Monolith". It was described in a blog post on his developer blog. The approach combines standard backface culling with internal face masking to optimize voxel rendering. [S1, unofficial]

## Sources
[S1] Graphic | Developerblog – Johannes Jendersie, https://jojendersie.de/category/graphic/, fetched 2026-10-24 (unofficial)
