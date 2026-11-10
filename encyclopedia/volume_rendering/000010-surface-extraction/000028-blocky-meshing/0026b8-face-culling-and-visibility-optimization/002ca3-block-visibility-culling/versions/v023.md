# Block Visibility Culling

Also called: Block Coordinate Change Detection, Block Rendering & Surface Orientation Culling, Camera-relative Block Culling, renderBlock

## Overview
Block Visibility Culling determines which voxel blocks require mesh updates or rendering based on the camera's position and rotation, avoiding unnecessary geometry generation when the viewing direction or position has not changed significantly [S1, unofficial]. It operates at the coarse block level by comparing current camera coordinates and primary viewing direction against previous frame values to skip index updates [S1, unofficial]. Unlike [Face Culling](8735-face-culling.md), which evaluates individual voxel faces against neighbors, this method evaluates entire blocks against the camera's global state [S1, unofficial]. It differs from [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md) by operating on blocks rather than individual faces within a block [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Camera world position $(C_x, C_y, C_z)$, camera rotation angles $(\theta_x, \theta_y)$, previous frame's camera block coordinates $(P_{bx}, P_{by}, P_{bz})$ and greatest viewing direction $(P_{dir})$, block size $B$, chunk origin $(ChunkX, ChunkY, ChunkZ)$, and block visibility flags [S1, unofficial][S2, unofficial].
*   **Output:** A set of blocks marked for rendering or mesh update; blocks matching previous state are skipped [S1, unofficial].

**Data Structures:**
*   `BlockFace` enum or flags: `YNeg`, `XPos`, `YPos`, `XNeg`, `ZNeg`, `ZPos` [S2, unofficial].
*   `visibleFaces`: Array or bitmask storing visibility status for each face of a block [S2, unofficial].
*   `blocksData`: Map or array storing indices for each block [S1, unofficial].

**Steps:**
1.  **Compute Camera Block Coordinates:**
    Calculate the integer block coordinates containing the camera by flooring its world position:
    $$ B_{cx} = \lfloor C_x / B \rfloor, \quad B_{cy} = \lfloor C_y / B \rfloor, \quad B_{cz} = \lfloor C_z / B \rfloor $$
    [S1, unofficial]

2.  **Compute View Vector:**
    Derive the view vector components $(V_x, V_y, V_z)$ from camera rotation angles $(\theta_x, \theta_y)$ [S1, unofficial].

3.  **Check for Change:**
    Compare current block coordinates $(B_{cx}, B_{cy}, B_{cz})$ and greatest viewing direction $(V_{dir})$ with previous frame's $(P_{bx}, P_{by}, P_{bz})$ and $(P_{dir})$ [S1, unofficial].
    *   If both match, skip the index update process for all blocks in the chunk [S1, unofficial].
    *   If they differ, proceed to evaluate block visibility [S1, unofficial].

4.  **Determine Surface Orientation (Per Block):**
    For each block at grid position $(x, y, z)$ within the chunk:
    *   Calculate world coordinates:
        $$ X = ChunkX + x \cdot B $$
        $$ Y = ChunkY + y \cdot B $$
        $$ Z_{bottom} = ChunkZ + z \cdot B $$
        $$ Z_{top} = Z_{bottom} + B $$
        [S2, unofficial]
    *   Determine surface orientation relative to camera $Z$ position:
        $$ O = \begin{cases} 1 & \text{if } C_z < Z_{bottom} \\ 2 & \text{if } C_z > Z_{top} \\ 0 & \text{otherwise} \end{cases} $$
        [S2, unofficial]
    *   Set rendering cycles:
        $$ Cycles = \begin{cases} 2 & \text{if } O \neq 0 \\ 1 & \text{otherwise} \end{cases} $$
        [S2, unofficial]

5.  **Transform to Camera Space:**
    *   Compute trigonometric values:
        $$ cs = \cos(\theta_y \cdot \pi / 180) $$
        $$ sn = \sin(\theta_y \cdot \pi / 180) $$
        [S2, unofficial]
    *   Calculate bottom corners relative to camera:
        $$ x_1 = X - C_x $$
        $$ y_1 = Y - C_y $$
        [S2, unofficial]
    *   Swap coordinates if orientation is top-facing ($O=1$):
        If $O == 1$, swap $x_1/x_2$ and $y_1/y_2$ [S2, unofficial].
    *   Project to screen space (partial):
        $$ WallX[w][0] = x_1 \cdot cs - y_1 \cdot sn $$
        [S2, unofficial]

6.  **Apply Face Visibility Flags:**
    Before rendering specific faces, check the `visibleFaces` flag for the corresponding `BlockFace` [S2, unofficial]:
    *   If $w=0$ (Bottom) and `!visibleFaces[YNeg]`, skip [S2, unofficial].
    *   If $w=1$ (Right) and `!visibleFaces[XPos]`, skip [S2, unofficial].
    *   If $w=2$ (Top) and `!visibleFaces[YPos]`, skip [S2, unofficial].
    *   If $w=3$ (Left) and `!visibleFaces[XNeg]`, skip [S2, unofficial] [S2, unofficial].

**Parameters:**
*   `surfaceScale`: Multiplier for floor/ceiling projection, typically 120 [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point comparisons for camera position changes should use a small epsilon to avoid jitter when the camera is stationary [G].
*   **Orientation Boundaries:** When $C_z$ is exactly equal to $Z_{bottom}$ or $Z_{top}$, the orientation logic must handle the boundary consistently to avoid flickering [S2, unofficial].

## Subtopics

### [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md)
Determines visibility of individual faces within a block based on normals and camera direction. Chosen when fine-grained face culling is needed within a block, unlike the coarse block-level check of the parent.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per frame for the change detection check. $O(N)$ for block evaluation if change is detected, where $N$ is the number of blocks in the chunk [S1, unofficial][S2, unofficial].
*   **Memory Complexity:** $O(1)$ for storing previous camera state. $O(N)$ for block visibility flags if stored per block [S2, unofficial].
*   **Performance:** Significantly reduces mesh updates when the camera is stationary or moving slowly [S1, unofficial]. Performance degrades if the camera moves frequently, requiring full block evaluation [S1, unofficial].
*   **Strengths:** Simple to implement, effective for static scenes or slow camera movement [S1, unofficial].
*   **Limitations:** Does not cull faces based on occlusion or neighbor voxels; relies on camera state only [S1, unofficial].

## Background
Originated in early voxel rendering implementations, such as those found in [exolands](https://github.com/kvxy/exolands) and [Voxel1993](https://github.com/VidFurlan/Voxel1993), to optimize rendering by avoiding unnecessary geometry generation based on camera movement [S1, unofficial][S2, unofficial].

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S2] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
