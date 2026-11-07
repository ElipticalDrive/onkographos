# Voxel Backface Culling and Internal Face Masking

Also called: Internal Face Masking, Voxel Backface Culling

## Overview
This method optimizes voxel mesh generation by rejecting faces that are either occluded by adjacent solid voxels (internal faces) or oriented away from the camera (backfaces). It distinguishes itself from sibling methods by performing visibility determination in a hybrid space: using projection-space geometry to determine front/back orientation and bit-packed integer masks to reject internal shared faces. This reduces the primitive count significantly compared to naive voxel rendering, where every voxel contributes up to 6 faces.

## How it works
The method operates on voxel chunks, typically of size $32^3$ voxels, to manage memory and enable large-scale culling [S1, unofficial]. It relies on a specific data layout to encode visibility and position efficiently.

### Inputs and Outputs
*   **Input**: A voxel chunk containing $N$ voxels. Each voxel is defined by its 8 corner positions in world space ($c\_vCorner_{ijk}$ where $i,j,k \in \{0,1\}$) and a 32-bit integer code ($vs\_out\_VoxelCode$) encoding face visibility and position.
*   **Output**: A stream of vertices and indices representing only the visible, non-internal faces of the voxels.

### Data Structures
The core optimization relies on a 32-bit integer used to encode both the position of the voxel and the visibility of its faces [S1, unofficial].

**Voxel Code Layout (32-bit integer):**
*   **Bits 0-5 (6 bits)**: Mask for each of the 6 sides of the voxel. A bit set to 1 indicates the face is potentially visible (not internal).
*   **Bits 6-30 (25 bits)**: Position information. Specifically, $3 \times 8$ bits are used for the position coordinates, allowing addressing of a $256^3$ volume space.
*   **Bits 31-32 (2 bits)**: Unused.

*Note: The source specifies 6 bits for the mask and $3 \times 8$ bits for position, totaling 30 bits, leaving 2 unused bits [S1, unofficial].*

### Parameters
*   **Chunk Size**: $32^3$ voxels per chunk. This size is chosen for efficient memory management, LOD support, and large-scale culling algorithms [S1, unofficial].
*   **Inverse Projection Diagonal**: A vector derived from the projection matrix to transform directions from projection space to view space.

### Steps
1.  **Frustum Culling**: First, reject voxels outside the viewing volume. Calculate the maximum length of the voxel corners and compare against the voxel's position in clip space ($vPos$). If $|vPos.z| > w$, $|vPos.x| > w$, or $|vPos.y| > w$ (where $w$ is the max corner length plus $vPos.w$), discard the voxel [S1, unofficial].
2.  **Transform View Direction**: Compute the view direction in projection space coordinates to facilitate the dot product test. This is done by multiplying the standard view direction by the inverse diagonal of the projection matrix:
    $$ v_{view} = v_{proj} \times \left( \frac{1}{m_{Projection}[0][0]}, \frac{1}{m_{Projection}[1][1]}, \frac{1}{m_{Projection}[2][2]} \right) $$
    [S1, unofficial]
3.  **Determine Face Orientation**: Calculate the dot product between the sum of specific voxel corners (transformed by the inverse projection diagonal) and the view direction. The specific corners used in the source example are $c\_vCorner000$ and $c\_vCorner110$ (representing a diagonal across the voxel in the XY plane, though the logic applies to determining front/back relative to the camera) [S1, unofficial].
    $$ \text{dot\_val} = \text{dot}\left( (c\_vCorner000.xyz + c\_vCorner110.xyz) \times c\_vInverseProjection.xyz, vViewDirection \right) $$
4.  **Process Front Face**: If $\text{dot\_val} > 0$, the face is considered a "front" face.
    *   Check if the internal masking bit `0x10` is set in $vs\_out\_VoxelCode[0]$.
    *   If set, emit the front face vertices: $c\_vCorner000, c\_vCorner100, c\_vCorner010, c\_vCorner110$.
    *   Compute lighting using the normal $(0, 0, -1)$ transformed by the world-view matrix [S1, unofficial].
5.  **Process Back Face**: If $\text{dot\_val} \le 0$, the face is considered a "back" face.
    *   Check if the internal masking bit `0x20` is set in $vs\_out\_VoxelCode[0]$.
    *   If set, emit the back face vertices: $c\_vCorner011, c\_vCorner111, c\_vCorner001, c\_vCorner101$.
    *   Compute lighting using the normal $(0, 0, 1)$ transformed by the world-view matrix [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_voxel_chunk(chunk, projection_matrix, world_view_matrix, view_direction):
    """
    Processes a chunk of voxels, applying backface culling and internal face masking.
    
    Args:
        chunk: List of voxel objects. Each voxel has:
            - corners: dict of 8 vec3s (c_vCorner000 to c_vCorner111)
            - voxel_code: int32 encoding visibility and position
        projection_matrix: 4x4 matrix
        world_view_matrix: 4x4 matrix
        view_direction: vec3 in view space
        
    Returns:
        List of emitted vertices and indices
    """
    # Precompute inverse projection diagonal
    inv_proj_diag = vec3(
        1.0 / projection_matrix[0][0],
        1.0 / projection_matrix[1][1],
        1.0 / projection_matrix[2][2]
    )
    
    # Transform view direction to projection space for dot product
    # Note: Source implies vViewDirection is used directly after scaling
    # The source code: dot((c_vCorner000.xyz+c_vCorner110.xyz)*c_vInverseProjection.xyz, vViewDirection)
    # This suggests vViewDirection is already in the correct space or the scaling is applied to the corner sum.
    # We follow the source formula exactly.
    
    emitted_vertices = []
    
    for voxel in chunk:
        c000 = voxel.corners['000']
        c110 = voxel.corners['110']
        
        # Frustum Culling Check
        # w = vPos.w + max(max(len(c000), len(c001)), max(len(c010), len(c110)))
        # For simplicity, assuming vPos is handled by the pipeline or passed in.
        # If vPos is not available, this step is skipped [G].
        # if abs(vPos.z) > w or abs(vPos.x) > w or abs(vPos.y) > w: continue
        
        # Calculate dot product for backface culling
        # Sum of corners scaled by inverse projection diagonal
        corner_sum = (c000.xyz + c110.xyz) * inv_proj_diag.xyz
        dot_val = dot(corner_sum, view_direction)
        
        voxel_code = voxel.voxel_code
        
        if dot_val > 0:
            # Front Face
            # Check bit 0x10 (bit 4)
            if (voxel_code & 0x10) != 0:
                # Emit front face vertices
                # Vertices: c_vCorner000, c_vCorner100, c_vCorner010, c_vCorner110
                # Normal: (0,0,-1) transformed by world-view matrix
                normal = normalize((vec4(0, 0, -1, 0) * world_view_matrix).xyz)
                # Lighting calculation would occur here
                # Emit vertices
                emit_vertex(c000)
                emit_vertex(voxel.corners['100'])
                emit_vertex(voxel.corners['010'])
                emit_vertex(c110)
                end_primitive()
        else:
            # Back Face
            # Check bit 0x20 (bit 5)
            if (voxel_code & 0x20) != 0:
                # Emit back face vertices
                # Vertices: c_vCorner011, c_vCorner111, c_vCorner001, c_vCorner101
                # Normal: (0,0,1) transformed by world-view matrix
                normal = normalize((vec4(0, 0, 1, 0) * world_view_matrix).xyz)
                # Lighting calculation would occur here
                # Emit vertices
                emit_vertex(voxel.corners['011'])
                emit_vertex(voxel.corners['111'])
                emit_vertex(voxel.corners['001'])
                emit_vertex(voxel.corners['101'])
                end_primitive()

def emit_vertex(pos):
    # Placeholder for actual vertex emission logic
    pass

def end_primitive():
    # Placeholder for primitive emission logic
    pass
```

### Edge Cases and Pitfalls
*   **Internal Faces**: Shared faces between two neighboring solid voxels are never visible. The masking bits (`0x10` and `0x20`) are specifically used to reject these invisible internal faces [S1, unofficial].
*   **Projection Space Limitations**: Direct backface culling in projection space is complex. The method uses the inverse projection diagonal trick to approximate the orientation test [S1, unofficial].
*   **Chunking**: For volumes larger than $256^3$, a chunked approach is necessary due to the 30-bit position encoding limit [S1, unofficial].

## Operating characteristics
*   **Performance**: On a GeForce 640M LE, the method draws approximately 32,368 voxels per millisecond [S1, unofficial].
*   **Complexity**: The per-voxel cost is constant time $O(1)$, involving a few dot products, bitwise operations, and conditional vertex emissions.
*   **Memory**: The 32-bit integer encoding is highly memory efficient, storing both position and visibility state in a single word per voxel [S1, unofficial].
*   **Strengths**: Significant reduction in primitive count by eliminating both backfaces and internal faces. Efficient memory usage due to bit-packing.
*   **Limitations**: Relies on a specific projection matrix structure for the inverse diagonal trick. Requires chunking for large volumes.

## Background
This technique was described by Johannes Jendersie in a blog post detailing the implementation of voxel rendering with geometry shaders. It addresses the inefficiency of rendering all voxel faces by introducing a hybrid culling strategy. The method is part of the "Monolith" voxel engine project.

## Sources
[S1] Graphic | Developerblog – Johannes Jendersie, https://jojendersie.de/category/graphic/, fetched 2026-10-24 (unofficial)
