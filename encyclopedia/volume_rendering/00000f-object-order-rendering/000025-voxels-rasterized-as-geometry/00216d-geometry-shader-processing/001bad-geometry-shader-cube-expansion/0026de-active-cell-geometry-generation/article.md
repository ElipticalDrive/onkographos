# Active Cell Geometry Generation

Also called: ActiveCellAndSOGS

## Overview
Active Cell Geometry Generation is a geometry shader technique that generates cube geometry for active voxels in a volume dataset [S1, unofficial]. It uses a flag volume to determine activity and expands a unit cube centered at the origin to the actual voxel size and world position [S1, unofficial]. It builds on [Geometry Shader Cube Expansion](1074-geometry-shader-cube-expansion.md) by specifically handling the generation of geometry for active cells within a pre-computed flag volume rather than generating geometry from raw triangle input [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A stream of primitives, where each primitive ID (`primID`) corresponds to a specific voxel cell in the volume grid [S1, unofficial].
*   **Input Data:** A 3D texture/sampler (`g_srvFlagVolume`) containing activity flags for each cell; a value of `1` indicates an active voxel [S1, unofficial].
*   **Constants:** `g_cf3VoxelReso` (total voxel resolution), `g_cf3HalfBrickSize` (half size of a voxel brick), `VOXEL_SIZE` (physical size of a single voxel), `VOXEL_BRICK_RATIO` (ratio dividing the volume into bricks), `VOXEL_NUM_Y` (total voxels in Y), `g_cuFlagVolSliceStep` and `g_cuFlagVolStripStep` (indexing steps), and `cb_mViewProj` (view-projection matrix) [S1, unofficial].
*   **Output:** A stream of vertices forming the 8 corners of a cube for each active voxel, transformed into projection space [S1, unofficial]. The output structure `PS_INPUT` contains `projPos` (`SV_POSITION`) and `Pos` (`TEXCOORD0`) [S1, unofficial].

**Data Structures:**
*   **Flag Volume:** A 3D texture where each texel stores an integer flag; `1` means active, other values mean inactive [S1, unofficial].
*   **Constant Buffer (`cbPerFrame`):** Contains camera matrices (`cb_mInvView`, `cb_mView`, `cb_mViewProj`), view position (`cb_f4ViewPos`), half window size (`cb_f2HalfWinSize`), window resolution (`cb_f2WinReso`), density range/isovalue (`cb_f2MinMaxDensity`), and NIU [S1, unofficial].

**Steps:**
1.  **Calculate 3D Index:** Derive the voxel coordinates $(u, v, w)$ from the primitive ID using integer division and modulo operations:
    $$
    \begin{aligned}
    u3Idx.z &= \text{primID} / g\_cuFlagVolSliceStep \\
    u3Idx.y &= (\text{primID} \% g\_cuFlagVolSliceStep) / g\_cuFlagVolStripStep \\
    u3Idx.x &= \text{primID} \% g\_cuFlagVolStripStep
    \end{aligned}
    $$
    [S1, unofficial]

2.  **Check Activity:** Sample the flag volume at the calculated index. If the value is not `1`, discard the primitive [S1, unofficial]:
    ```python
    active = g_srvFlagVolume[u3Idx]
    if active != 1:
        return [] # Discard primitive
    ```

3.  **Invert Y-Axis:** Adjust the Y-index to match the coordinate system, accounting for the brick ratio:
    $$
    u3Idx.y = \frac{VOXEL\_NUM\_Y}{VOXEL\_BRICK\_RATIO} - 1 - u3Idx.y
    $$
    [S1, unofficial]

4.  **Calculate World Space Offset:** Compute the center offset of the voxel in world space:
    $$
    f4CellCenterOffset = \left( (u3Idx - \frac{g\_cf3VoxelReso \times 0.5}{VOXEL\_BRICK\_RATIO} + 0.5) \times VOXEL\_SIZE \times VOXEL\_BRICK\_RATIO, 0.0 \right)
    $$
    [S1, unofficial]

5.  **Define Half-Size:** Determine the half-extent of the cube. For solid rendering, use $1.2 \times$ the half-brick size; for wireframe, use the standard half-brick size:
    $$
    f4HalfCellSize = \begin{cases} 
    (g\_cf3HalfBrickSize \times 1.2, 1.0) & \text{solid} \\
    (g\_cf3HalfBrickSize, 1.0) & \text{wireframe}
    \end{cases}
    $$
    [S1, unofficial]

6.  **Generate Vertices:** For each of the 8 corners of a unit cube (e.g., $(\pm 1, \pm 1, \pm 1)$), compute the world position and transform to projection space:
    $$
    \begin{aligned}
    output.Pos &= \text{unit\_corner} \times f4HalfCellSize + f4CellCenterOffset \\
    output.projPos &= \text{mul}(output.Pos, cb\_mViewProj)
    \end{aligned}
    $$
    [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def GenerateActiveCellGeometry(primID, g_cuFlagVolSliceStep, g_cuFlagVolStripStep, 
                               VOXEL_NUM_Y, VOXEL_BRICK_RATIO, g_cf3VoxelReso, 
                               VOXEL_SIZE, g_cf3HalfBrickSize, cb_mViewProj, g_srvFlagVolume):
    # Step 1: Calculate 3D index from primitive ID
    idx_z = primID // g_cuFlagVolSliceStep
    idx_y = (primID % g_cuFlagVolSliceStep) // g_cuFlagVolStripStep
    idx_x = primID % g_cuFlagVolStripStep
    u3Idx = (idx_x, idx_y, idx_z)

    # Step 2: Check if cell is active
    if g_srvFlagVolume[u3Idx] != 1:
        return [] # Discard primitive

    # Step 3: Invert Y-axis index
    idx_y = (VOXEL_NUM_Y // VOXEL_BRICK_RATIO) - 1 - idx_y
    u3Idx = (idx_x, idx_y, idx_z)

    # Step 4: Calculate world space offset to cell center
    reso_scaled = tuple(r * 0.5 / VOXEL_BRICK_RATIO for r in g_cf3VoxelReso)
    center_offset_vec = tuple((u3Idx[i] - reso_scaled[i] + 0.5) * VOXEL_SIZE * VOXEL_BRICK_RATIO for i in range(3))
    f4CellCenterOffset = (*center_offset_vec, 0.0)

    # Step 5: Define half-size of the cell (solid rendering)
    half_brick = g_cf3HalfBrickSize
    f4HalfCellSize = tuple(h * 1.2 for h in half_brick) + (1.0,)

    # Step 6: Generate vertices for the 8 corners
    unit_corners = [
        (-1, -1, -1), (-1, -1, 1), (-1, 1, -1), (-1, 1, 1),
        (1, -1, -1), (1, -1, 1), (1, 1, -1), (1, 1, 1)
    ]

    output_vertices = []
    for corner in unit_corners:
        # Calculate world position
        pos = tuple(corner[i] * f4HalfCellSize[i] + f4CellCenterOffset[i] for i in range(3)) + (f4CellCenterOffset[3],)

        # Transform to projection space (row vector * matrix)
        proj = tuple(sum(pos[i] * cb_mViewProj[i][j] for i in range(4)) for j in range(4))

        vertex = {
            'projPos': proj,
            'Pos': pos
        }
        output_vertices.append(vertex)

    return output_vertices
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The Y-axis inversion formula assumes a specific coordinate system orientation where Y is inverted per brick [S1, unofficial].
*   **Precision:** Floating-point arithmetic in the offset calculation may introduce small errors, potentially causing gaps or overlaps between adjacent voxel cubes [G].
*   **Inactive Voxels:** Primitives corresponding to inactive voxels are discarded, which may result in non-contiguous primitive streams if not handled by the rasterizer or subsequent stages [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of primitives (voxels) in the input stream, as each primitive is processed independently in the geometry shader [G].
*   **Memory Complexity:** $O(1)$ per primitive, plus the storage for the flag volume texture [G].
*   **Performance:** Performance depends on the number of active voxels and the complexity of the geometry shader [S1, unofficial]. The use of a flag volume allows for efficient culling of inactive voxels, reducing the number of cubes generated [S1, unofficial].
*   **Strengths:** Simple implementation and efficient culling of empty space [S1, unofficial].
*   **Limitations:** Generates a large number of vertices (8 per active voxel), which can be expensive for high-resolution volumes [G].
*   **When to Use:** Suitable for interactive applications where the volume resolution is moderate and the number of active voxels is sparse [G].
*   **When Not to Use:** Not suitable for high-resolution volumes with dense active voxels due to vertex generation overhead [G].

## Background
The method is specific to the Direct3D 11 implementation described in the source [S1, unofficial].

## Sources
[S1] pengliu916/DynamicVolumeRayCasting, https://github.com/pengliu916/DynamicVolumeRayCasting, fetched 2026-10-26 (unofficial)
