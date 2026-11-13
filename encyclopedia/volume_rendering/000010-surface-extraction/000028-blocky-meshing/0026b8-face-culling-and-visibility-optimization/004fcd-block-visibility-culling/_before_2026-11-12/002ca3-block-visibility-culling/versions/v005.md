# Block Visibility Culling

Also called: Block Coordinate Change Detection, Block Rendering & Surface Orientation Culling, Camera-relative Block Culling, renderBlock

## Overview
Block Visibility Culling determines which voxel blocks require mesh updates or rendering based on the camera's position and rotation, avoiding unnecessary geometry generation when the camera has not moved significantly or changed its primary viewing direction [S1, unofficial]. Unlike sibling methods that operate at the fine-grained voxel face level [6723-voxel-face-visibility-check.md] or use coverage masks [3397-face-coverage-mask-culling.md], this method operates at the coarse block level, leveraging camera-relative orientation to cull entire block faces before they are processed into the mesh [S1, unofficial]. It serves as a pre-filter for the parent [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md) by reducing the set of candidate blocks.

## How it works
**Inputs:** Camera world position $(P_x, P_y, P_z)$, camera rotation angle $\theta$ (in degrees), block grid dimensions (chunk coordinates, block size), and previous frame's camera state (block coordinates, greatest viewing direction).
**Outputs:** A set of blocks marked for rendering or mesh update; invisible blocks are skipped.

**Data Structures:**
*   `BlockFace`: An enum or bitfield representing the six faces of a block: `YNeg`, `XPos`, `YPos`, `XNeg`, `ZNeg`, `ZPos` [S2, unofficial].
*   `visibleFaces`: An array or bitmask associated with each block, storing the visibility state of its faces [S2, unofficial].
*   `prevBlockCoords`: Integer tuple $(blockX_{prev}, blockY_{prev}, blockZ_{prev})$ storing the camera's block coordinates from the previous frame [S1, unofficial].
*   `prevGreatestDir`: The dominant axis of the camera's view vector from the previous frame [S1, unofficial].

**Steps:**
1.  **Compute Camera Block Coordinates:**
    Calculate the integer block coordinates containing the camera by flooring the world position:
    $$
    blockX_{curr} = \lfloor P_x / BLOCK\_SIZE \rfloor
    $$
    $$
    blockY_{curr} = \lfloor P_y / BLOCK\_SIZE \rfloor
    $$
    $$
    blockZ_{curr} = \lfloor P_z / BLOCK\_SIZE \rfloor
    $$
    [S1, unofficial]

2.  **Check for Redundant Updates:**
    Compute the current view vector components $(v_x, v_y, v_z)$ from the camera's rotation angles (e.g., $v_x = \cos(\theta)$, $v_y = \sin(\theta)$ in 2D, or extended to 3D spherical coordinates) [S1, unofficial]. Determine the `greatestDir` as the axis with the largest absolute component in the view vector [S1, unofficial].
    If $(blockX_{curr}, blockY_{curr}, blockZ_{curr}) == (blockX_{prev}, blockY_{prev}, blockZ_{prev})$ AND `greatestDir` == `prevGreatestDir`, skip the index update process for this frame [S1, unofficial].

3.  **Transform to Camera Space:**
    For each block in the visible chunk or relevant range:
    Calculate the block's world boundaries:
    $$
    blockX = chunkX \times CHUNK\_SIZE + x \times BLOCK\_SIZE
    $$
    $$
    blockY = chunkY \times CHUNK\_SIZE + y \times BLOCK\_SIZE
    $$
    $$
    blockZ_{bottom} = chunkZ \times CHUNK\_SIZE + z \times BLOCK\_SIZE
    $$
    $$
    blockZ_{top} = blockZ_{bottom} + BLOCK\_SIZE
    $$
    [S2, unofficial]
    
    Calculate the relative position of the block's bottom corners to the camera:
    $$
    x_1 = x_{bottom1}[w] - P_x
    $$
    $$
    y_1 = y_{bottom1}[w] - P_y
    $$
    [S2, unofficial]
    
    Compute the rotation matrix components:
    $$
    cs = \cos(\theta \times \pi / 180)
    $$
    $$
    sn = \sin(\theta \times \pi / 180)
    $$
    [S2, unofficial]
    
    Transform to camera-relative coordinates:
    $$
    wallX[w][0] = x_1 \times cs - y_1 \times sn
    $$
    [S2, unofficial]

4.  **Determine Surface Orientation:**
    Determine the surface orientation relative to the camera's vertical position:
    $$
    orientation = 
    \begin{cases} 
    1 & \text{if } P_z < blockZ_{bottom} \quad (\text{Looking Up}) \\
    2 & \text{if } P_z > blockZ_{top} \quad (\text{Looking Down}) \\
    0 & \text{otherwise} \quad (\text{Level})
    \end{cases}
    $$
    [S2, unofficial]
    
    Set the number of cycles for face processing:
    $$
    cycles = 
    \begin{cases} 
    2 & \text{if } orientation \neq 0 \\
    1 & \text{otherwise}
    \end{cases}
    $$
    [S2, unofficial]

5.  **Cull Invisible Faces:**
    Iterate through the block's faces (e.g., $w$ from 0 to 3 for side walls) [S2, unofficial]. Before rendering or adding to the mesh, check the `visibleFaces` flag for the specific face [S2, unofficial]:
    *   If $w=0$ and `!block->visibleFaces[BlockFace::YNeg]`, continue.
    *   If $w=1$ and `!block->visibleFaces[BlockFace::XPos]`, continue.
    *   If $w=2$ and `!block->visibleFaces[BlockFace::YPos]`, continue.
    *   If $w=3$ and `!block->visibleFaces[BlockFace::XNeg]`, continue.
    [S2, unofficial]
    
    If the face is visible, project the vertices and add to the mesh. If the orientation is 1 (Looking Up), swap $x_1/x_2$ and $y_1/y_2$ to correct the projection [S2, unofficial].

**Parameters:**
*   `surfaceScale`: Multiplier for floor/ceiling projection, typically 120 [S2, unofficial].
*   `BLOCK_SIZE`: The size of a single voxel block in world units.

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point comparisons for camera position should use a small epsilon to prevent flickering when the camera is exactly on a block boundary [G].
*   **Boundary Conditions:** When $xPos2 - xPos1 = 0$, avoid division by zero in horizontal step calculations by setting the denominator to 1 [S2, unofficial].
*   **Degenerate Input:** If the camera is inside a block, the orientation logic may need special handling to ensure all faces are considered visible or culled appropriately based on the interior view [G].

## Subtopics

### [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md)
Operates at the fine-grained face level within a block, evaluating geometric relationships between face normals and the reference direction. Choose this when block-level culling is insufficient and per-face visibility is required.

### [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md)
Uses projection masks to determine visibility. Choose this when occlusion from other blocks is a primary concern.

### [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md)
Focuses on removing backfacing triangles. Choose this for general mesh optimization after block culling.

## Operating characteristics
*   **Time Complexity:** $O(1)$ for the coordinate check and orientation determination per block. The culling check is $O(1)$ per face. The overall cost is proportional to the number of blocks in the view frustum, but significantly reduced by skipping updates when the camera state is unchanged [S1, unofficial].
*   **Memory Complexity:** $O(N)$ where $N$ is the number of blocks, storing the `visibleFaces` state for each block [S2, unofficial].
*   **Performance:** Significantly reduces mesh generation time in static scenes or when the camera moves slowly, as redundant index updates are skipped [S1, unofficial].
*   **Strengths:** Simple to implement, low overhead, effective for large-scale voxel worlds with limited camera movement.
*   **Limitations:** Does not account for occlusion by other blocks (only camera-relative orientation). Requires maintaining previous frame state.
*   **When to use:** Ideal for real-time applications where camera movement is the primary driver of visibility changes, and occlusion is handled by other methods (e.g., depth testing or chunk-level culling).
*   **When not to use:** In scenes with rapid camera rotation or where occlusion is the dominant factor in visibility.

## Background
Originating from early voxel rendering implementations in game development, this method was popularized by projects like Exolands and Voxel1993, which optimized rendering by leveraging camera-relative block coordinates and orientation flags [S1, unofficial][S2, unofficial]. It builds on the principle that faces pointing away from the camera or obscured by the camera's position relative to the block's vertical extent can be safely ignored.

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S2] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
