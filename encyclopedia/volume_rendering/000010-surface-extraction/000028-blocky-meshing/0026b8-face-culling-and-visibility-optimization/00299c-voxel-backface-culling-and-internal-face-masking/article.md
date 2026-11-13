# Voxel Backface Culling and Internal Face Masking

Also called: Internal Face Masking, Voxel Backface Culling

## Overview
This method optimizes voxel mesh generation by rejecting faces that are either geometrically internal to the solid or oriented away from the camera in projection space. It builds on [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md) by adding a visibility test against the view direction and a bitmask-based rejection of shared internal faces. Unlike sibling methods that rely on block placement or coverage masks, this approach operates directly on voxel corners and normals within a geometry shader or post-vertex stage, using a compact 32-bit integer to encode both position and face visibility state.

## How it works
**Inputs:**
- Voxel corner positions in world space: $c_{vCorner000}, c_{vCorner100}, c_{vCorner010}, c_{vCorner110}, c_{vCorner001}, c_{vCorner101}, c_{vCorner011}, c_{vCorner111}$ [S1, unofficial].
- View direction vector $v_{ViewDirection}$ in world space [S1, unofficial].
- Projection matrix $m_{Projection}$ [S1, unofficial].
- World-view matrix $c_{mWorldView}$ [S1, unofficial].
- A 32-bit integer `vs_out_VoxelCode[0]` encoding face visibility masks and position [S1, unofficial].

**Outputs:**
- Vertex stream for visible, front-facing, non-internal faces [S1, unofficial].

**Data Structures:**
- `vs_out_VoxelCode[0]`: A 32-bit integer.
  - Bits 0-5: Mask for each of the 6 sides (front/back, left/right, top/bottom). Specifically, bit 4 (`0x10`) controls the front face (Z-) and bit 5 (`0x20`) controls the back face (Z+) [S1, unofficial].
  - Bits 6-31: Position information (3 * 8 bits for X, Y, Z coordinates) [S1, unofficial].
  - Bits 32-33: Unused [S1, unofficial].
  - This layout supports addressing up to $256^3$ positions [S1, unofficial].

**Parameters:**
- Chunk size: $32^3$ voxels per chunk [S1, unofficial].
- Typical hardware: Tested on GeForce 640M LE [S1, unofficial].

**Steps:**
1. **Bounding Box Check:** Reject voxels outside the viewing volume. Calculate $w = v_{Pos}.w + \max(\max(\|c_{vCorner000}\|, \|c_{vCorner001}\|), \max(\|c_{vCorner010}\|, \|c_{vCorner111}\|))$. If $|v_{Pos}.z| > w$ or $|v_{Pos}.x| > w$ or $|v_{Pos}.y| > w$, discard the voxel [S1, unofficial].
2. **Transform View Direction to Projection Space:** Compute the inverse projection diagonal vector $v_{InverseProjection} = (1/m_{Projection}[0][0], 1/m_{Projection}[1][1], 1/m_{Projection}[2][2])$. Transform the view direction: $v_{ViewProj} = v_{ViewDirection} * v_{InverseProjection}$ [S1, unofficial].
3. **Determine Face Orientation:** Calculate the dot product of the sum of two opposite voxel corners (transformed by the inverse projection diagonal) and the view direction:
   $$ D = \text{dot}((c_{vCorner000}.xyz + c_{vCorner110}.xyz) * v_{InverseProjection}.xyz, v_{ViewDirection}) $$
   If $D > 0$, the front face is considered front-facing; otherwise, the back face is considered front-facing [S1, unofficial].
4. **Internal Face Masking and Emission:**
   - **If Front-Facing ($D > 0$):** Check if bit `0x10` is set in `vs_out_VoxelCode[0]`. If set, emit the front face vertices: $c_{vCorner000}, c_{vCorner100}, c_{vCorner010}, c_{vCorner110}$. Compute the normal as $n = \text{normalize}((vec4(0,0,-1,0) * c_{mWorldView}).xyz)$ and apply lighting [S1, unofficial].
   - **If Back-Facing ($D \le 0$):** Check if bit `0x20` is set in `vs_out_VoxelCode[0]`. If set, emit the back face vertices: $c_{vCorner011}, c_{vCorner111}, c_{vCorner001}, c_{vCorner101}$. Compute the normal as $n = \text{normalize}((vec4(0,0,1,0) * c_{mWorldView}).xyz)$ and apply lighting [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_voxel(c_vCorner000, c_vCorner100, c_vCorner010, c_vCorner110,
                  c_vCorner001, c_vCorner101, c_vCorner011, c_vCorner111,
                  vPos, vViewDirection, mProjection, c_mWorldView, vs_out_VoxelCode):
    # Step 1: Bounding Box Check
    w = vPos.w + max(max(length(c_vCorner000), length(c_vCorner001)),
                     max(length(c_vCorner010), length(c_vCorner111)))
    if abs(vPos.z) > w or abs(vPos.x) > w or abs(vPos.y) > w:
        return

    # Step 2: Transform View Direction
    inv_proj = vec3(1.0 / mProjection[0][0], 1.0 / mProjection[1][1], 1.0 / mProjection[2][2])
    v_view_proj = vViewDirection * inv_proj

    # Step 3: Determine Face Orientation
    corner_sum = (c_vCorner000.xyz + c_vCorner110.xyz) * inv_proj
    dot_product = dot(corner_sum, vViewDirection)

    # Step 4: Internal Face Masking and Emission
    if dot_product > 0:
        # Front face check
        if (vs_out_VoxelCode[0] & 0x10) != 0:
            vNormal = normalize((vec4(0, 0, -1, 0) * c_mWorldView).xyz)
            # Emit front face vertices
            emit_vertex(c_vCorner000 + vPos)
            emit_vertex(c_vCorner100 + vPos)
            emit_vertex(c_vCorner010 + vPos)
            emit_vertex(c_vCorner110 + vPos)
            # Apply lighting with vNormal
    else:
        # Back face check
        if (vs_out_VoxelCode[0] & 0x20) != 0:
            vNormal = normalize((vec4(0, 0, 1, 0) * c_mWorldView).xyz)
            # Emit back face vertices
            emit_vertex(c_vCorner011 + vPos)
            emit_vertex(c_vCorner111 + vPos)
            emit_vertex(c_vCorner001 + vPos)
            emit_vertex(c_vCorner101 + vPos)
            # Apply lighting with vNormal
```

**Edge Cases and Pitfalls:**
- Shared faces between two neighboring voxels are never visible and must be masked using bits `0x10` and `0x20` [S1, unofficial].
- The method assumes a specific projection matrix structure for the inverse diagonal transformation [S1, unofficial].
- The bounding box check uses a simplified distance metric that may not be robust for all camera configurations [S1, unofficial].

## Operating characteristics
- **Performance:** Draws 32,368 voxels per millisecond on a GeForce 640M LE [S1, unofficial].
- **Memory:** Uses a 32-bit integer per voxel for position and mask, supporting $256^3$ positions [S1, unofficial].
- **Scaling:** Requires a chunked approach (e.g., $32^3$ chunks) for larger models to manage memory and enable LOD/culling [S1, unofficial].
- **Strengths:** Efficient rejection of internal and back-facing faces in a single pass; compact data representation [S1, unofficial].
- **Limitations:** Geometry shader performance can be a bottleneck for large datasets; relies on specific projection matrix assumptions [S1, unofficial].

## Background
This technique was developed by Johannes Jendersie for the "Monolith" voxel engine, as described in his graphic developer blog [S1, unofficial]. It addresses the complexity of drawing proper voxel beams and grids, which are useful for lighting effects like god-rays or laser weapons [S1, unofficial]. The method was tested on a GeForce 640M LE and aims to improve rendering rates over simple line or wireframe approaches [S1, unofficial].

## Sources
[S1] Graphic | Developerblog – Johannes Jendersie, https://jojendersie.de/category/graphic/, fetched 2026-10-24 (unofficial)
