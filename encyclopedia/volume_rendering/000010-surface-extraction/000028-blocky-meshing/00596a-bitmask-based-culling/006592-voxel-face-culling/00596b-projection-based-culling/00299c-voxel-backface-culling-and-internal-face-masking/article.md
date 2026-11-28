# Voxel Backface Culling and Internal Face Masking

Also called: Internal Face Masking, Voxel Backface Culling

## Overview
Voxel Backface Culling and Internal Face Masking is a technique to reject invisible faces in voxel rendering by determining front/back orientation in projection space and masking faces shared between neighboring voxels [S1, unofficial]. It sets itself apart from sibling methods by operating directly on voxel primitives in the geometry shader stage, using a specific projection-space dot product to orient faces and a bitmask to eliminate internal geometry, rather than relying on mesh connectivity or external depth buffers [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input**: A voxel defined by eight corner positions ($c_{vCorner000}$ to $c_{vCorner111}$), a view direction vector ($v_{ViewDirection}$), a world-view matrix ($c_{mWorldView}$), and a 32-bit integer encoding visibility masks and position ($vs\_out\_VoxelCode[0]$) [S1, unofficial].
*   **Output**: Vertices emitted for the visible front or back face of the voxel, with associated lighting and screen-space positions [S1, unofficial].

**Data Structures**
The visibility state is encoded in a single 32-bit integer ($vs\_out\_VoxelCode[0]$) [S1, unofficial]:
*   Bits 0-5: Mask for the six sides of the voxel (1 bit per side).
*   Bits 6-23: 3 × 8 bits for the 3D position (allowing addressing of $256^3$ positions).
*   Bits 24-31: Unused [S1, unofficial].

**Steps**
1.  **Calculate Inverse Projection Diagonal**: Compute the vector $v_{InverseProjection}$ using the diagonal elements of the projection matrix $mProjection$:
    $$ v_{InverseProjection} = \left( \frac{1}{mProjection[0][0]}, \frac{1}{mProjection[1][1]}, \frac{1}{mProjection[2][2]} \right) $$
    This vector transforms directions from projection space to view space [S1, unofficial].

2.  **Determine Face Orientation**: Calculate the dot product of the sum of two opposite voxel corners (transformed by the inverse projection diagonal) and the view direction [S1, unofficial]:
    $$ D = \text{dot}\left( (c_{vCorner000}.xyz + c_{vCorner110}.xyz) * v_{InverseProjection}.xyz, v_{ViewDirection} \right) $$
    *   If $D > 0$, the front face is oriented towards the viewer [S1, unofficial].
    *   If $D \le 0$, the back face is oriented towards the viewer [S1, unofficial].

3.  **Check Internal Masking and Emit**:
    *   **Front Face Case** ($D > 0$): Check if bit `0x10` (bit 4) is set in $vs\_out\_VoxelCode[0]$ [S1, unofficial].
        *   If set, emit the front face vertices: $c_{vCorner000}, c_{vCorner100}, c_{vCorner010}, c_{vCorner110}$ [S1, unofficial].
        *   Compute lighting using the normal $n = (0, 0, -1)$ transformed by the world-view matrix: $v_{Normal} = \text{normalize}((vec4(0,0,-1,0) * c_{mWorldView}).xyz)$ [S1, unofficial].
    *   **Back Face Case** ($D \le 0$): Check if bit `0x20` (bit 5) is set in $vs\_out\_VoxelCode[0]$ [S1, unofficial].
        *   If set, emit the back face vertices: $c_{vCorner011}, c_{vCorner111}, c_{vCorner001}, c_{vCorner101}$ [S1, unofficial].
        *   Compute lighting using the normal $n = (0, 0, 1)$ transformed by the world-view matrix: $v_{Normal} = \text{normalize}((vec4(0,0,1,0) * c_{mWorldView}).xyz)$ [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_voxel(c_vCorner000, c_vCorner001, c_vCorner010, c_vCorner011,
                  c_vCorner100, c_vCorner101, c_vCorner110, c_vCorner111,
                  vs_out_VoxelCode, vViewDirection, c_mWorldView, mProjection):
    # Step 1: Calculate inverse projection diagonal
    inv_proj = (1.0 / mProjection[0][0], 1.0 / mProjection[1][1], 1.0 / mProjection[2][2])

    # Step 2: Determine orientation
    corner_sum = (
        (c_vCorner000[0] + c_vCorner110[0]) * inv_proj[0],
        (c_vCorner000[1] + c_vCorner110[1]) * inv_proj[1],
        (c_vCorner000[2] + c_vCorner110[2]) * inv_proj[2]
    )
    dot_product = (corner_sum[0] * vViewDirection[0] +
                   corner_sum[1] * vViewDirection[1] +
                   corner_sum[2] * vViewDirection[2])

    emitted_vertices = []

    def transform_vec4(v, mat):
        # Row-vector multiplication: v * mat
        return (
            v[0]*mat[0][0] + v[1]*mat[1][0] + v[2]*mat[2][0] + v[3]*mat[3][0],
            v[0]*mat[0][1] + v[1]*mat[1][1] + v[2]*mat[2][1] + v[3]*mat[3][1],
            v[0]*mat[0][2] + v[1]*mat[1][2] + v[2]*mat[2][2] + v[3]*mat[3][2],
            v[0]*mat[0][3] + v[1]*mat[1][3] + v[2]*mat[2][3] + v[3]*mat[3][3]
        )

    def emit_vertex(v):
        emitted_vertices.append(v)

    if dot_product > 0:
        # Front face
        if (vs_out_VoxelCode[0] & 0x10) != 0:
            normal_vec = transform_vec4((0, 0, -1, 0), c_mWorldView)
            normal = (normal_vec[0], normal_vec[1], normal_vec[2]) / (normal_vec[0]**2 + normal_vec[1]**2 + normal_vec[2]**2)**0.5
            emit_vertex(c_vCorner000)
            emit_vertex(c_vCorner100)
            emit_vertex(c_vCorner010)
            emit_vertex(c_vCorner110)
            # Lighting computation is shader-specific and omitted [G]
    else:
        # Back face
        if (vs_out_VoxelCode[0] & 0x20) != 0:
            normal_vec = transform_vec4((0, 0, 1, 0), c_mWorldView)
            normal = (normal_vec[0], normal_vec[1], normal_vec[2]) / (normal_vec[0]**2 + normal_vec[1]**2 + normal_vec[2]**2)**0.5
            emit_vertex(c_vCorner011)
            emit_vertex(c_vCorner111)
            emit_vertex(c_vCorner001)
            emit_vertex(c_vCorner101)
            # Lighting computation is shader-specific and omitted [G]
    return emitted_vertices
```

**Parameters**
*   **Chunk Size**: $32^3$ voxels per chunk. This size is used for efficient memory management, LOD, and large-scale culling algorithms [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Shared Faces**: Faces shared between two neighboring voxels are never visible. The masking bits (`0x10` and `0x20`) are specifically used to reject these invisible internal faces [S1, unofficial].
*   **Projection Space Limitations**: Backface culling cannot be done directly in projection space without the inverse projection diagonal trick; the direction must be transformed to view space first [S1, unofficial].
*   **Frustum Culling**: Voxels outside the viewing volume should be rejected before this step. The source implements this by computing $w = vPos.w + \max(\max(\|c_{vCorner000}\|, \|c_{vCorner001}\|), \max(\|c_{vCorner010}\|, \|c_{vCorner111}\|))$ and rejecting the voxel if $\|vPos.z\| > w$, $\|vPos.x\| > w$, or $\|vPos.y\| > w$ [S1, unofficial].

## Operating characteristics
*   **Performance**: On a GeForce 640M LE, the method draws 32,368 voxels per millisecond [S1, unofficial].
*   **Complexity**: The method adds a constant-time dot product and bitwise check per voxel in the geometry shader [G].
*   **Strengths**: Efficiently culls internal geometry that would otherwise be rendered and discarded by the depth test, avoiding the overhead of mesh connectivity analysis or depth buffer lookups [G].
*   **Limitations**: Requires a specific projection matrix structure to compute the inverse diagonal [S1, unofficial]. Relies on pre-computed voxel codes for masking [S1, unofficial]. Performance is limited by geometry shader throughput on the target hardware [G].

## Background
This technique was described by Johannes Jendersie in a blog post on his developer blog, focusing on voxel rendering optimization using geometry shaders [S1, unofficial]. It was developed as part of a voxel beam/grid rendering system to address geometry shader throughput limitations by reducing the number of primitives sent to the rasterizer [S1, unofficial]. The approach leverages the regular structure of voxels to simplify visibility testing [S1, unofficial].

## Sources
[S1] Voxel Backface Culling and Internal Face Masking, https://jojendersie.de/category/graphic/, fetched 2026-10-24 (unofficial)
[S2] Forum archive / Ken Silverman / Real-time voxel raytracing, in development.., https://www.jonof.id.au/forum/topic-1433.html, fetched 2026-10-26 (unofficial)
