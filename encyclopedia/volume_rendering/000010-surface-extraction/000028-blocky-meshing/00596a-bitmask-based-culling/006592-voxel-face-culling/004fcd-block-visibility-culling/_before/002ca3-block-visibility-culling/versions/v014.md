# Block Visibility Culling

Also called: Block Coordinate Change Detection, Block Rendering & Surface Orientation Culling, Camera-relative Block Culling, renderBlock

## Overview
Block Visibility Culling determines which voxel blocks require mesh updates or rendering based on the camera's position and rotation, avoiding unnecessary geometry generation when the camera has not moved significantly or changed its primary viewing direction [S1, unofficial][S2, unofficial]. It operates at the coarse block level by comparing current camera block coordinates and viewing direction against previous frame values, serving as a high-level optimization that complements finer-grained face culling [S1, unofficial]. Unlike [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md), which evaluates individual face normals, this method skips entire block processing if the camera's relative block position and orientation remain stable [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Camera world position $(P_x, P_y, P_z)$, camera rotation angle $\theta$ (in degrees), previous frame's camera block coordinates $(B_{x,prev}, B_{y,prev}, B_{z,prev})$, previous frame's greatest viewing direction indicator $G_{prev}$, block dimensions $BLOCK\_SIZE$, and block visibility flags (e.g., `visibleFaces` array).
*   **Output:** Boolean flag indicating whether the block mesh needs regeneration; if regenerated, the visible faces are determined for rendering.

**Data Structures:**
*   **Block Visibility Flags:** An array or enum `BlockFace` with 6 flags: `YNeg`, `XPos`, `YPos`, `XNeg`, `ZNeg`, `ZPos` [S2, unofficial]. Each flag indicates whether the corresponding face of the block is visible and should be rendered.
*   **Camera State:** Stores integer block coordinates and a direction indicator (e.g., `greatestDir`) from the previous frame.

**Steps:**
1.  **Compute Camera Block Coordinates:** Calculate the integer block coordinates containing the camera by flooring its world position:
    $$ B_x = \lfloor P_x / BLOCK\_SIZE \rfloor $$
    $$ B_y = \lfloor P_y / BLOCK\_SIZE \rfloor $$
    $$ B_z = \lfloor P_z / BLOCK\_SIZE \rfloor $$
    [S1, unofficial]

2.  **Determine Viewing Direction Indicator:** Compute the view vector components $(v_x, v_y, v_z)$ from the camera's rotation angles $(r_x, r_y)$. Identify the axis with the greatest magnitude in the view vector to determine the primary viewing direction $G$ [S1, unofficial].

3.  **Change Detection:** Compare the current camera block coordinates $(B_x, B_y, B_z)$ and the greatest viewing direction indicator $G$ with the previous frame's values $(B_{x,prev}, B_{y,prev}, B_{z,prev}, G_{prev})$. If both the block coordinates and the direction indicator match, skip the index update process for this block [S1, unofficial].

4.  **Surface Orientation Calculation (if update required):**
    *   Calculate trigonometric values: $cs = \cos(\theta \cdot \pi / 180)$, $sn = \sin(\theta \cdot \pi / 180)$ [S1, unofficial].
    *   For each block at chunk-relative coordinates $(x, y, z)$, compute world coordinates:
        $$ blockX = chunkX + x \cdot BLOCK\_SIZE $$
        $$ blockY = chunkY + y \cdot BLOCK\_SIZE $$
        $$ blockZ_{Bottom} = chunkZ + z \cdot BLOCK\_SIZE $$
        $$ blockZ_{Top} = blockZ_{Bottom} + BLOCK\_SIZE $$
        [S1, unofficial]
    *   Determine surface orientation relative to the camera's Z position:
        $$ orientation = \begin{cases} 1 & \text{if } P_z < blockZ_{Bottom} \\ 2 & \text{if } P_z > blockZ_{Top} \\ 0 & \text{otherwise} \end{cases} $$
        [S1, unofficial]
    *   Set the number of cycles for face processing: $cycles = 2$ if $orientation \neq 0$, else $1$ [S1, unofficial].

5.  **Face Visibility Check:** For each of the six faces of the block, check the corresponding flag in the `visibleFaces` array. If the flag is false, skip rendering that face [S2, unofficial]. For example, if processing the bottom face ($w=0$), check `block->visibleFaces[BlockFace::YNeg]` [S2, unofficial].

6.  **Projection and Rendering (if visible):**
    *   Transform block corners to camera space:
        $$ x_1 = x_{Bottom1}[w] - P_x $$
        $$ y_1 = y_{Bottom1}[w] - P_y $$
        [S1, unofficial]
    *   Swap $x_1/x_2$ and $y_1/y_2$ if $orientation == 1$ [S1, unofficial].
    *   Project to screen space using the calculated $cs$ and $sn$ values:
        $$ wallX[w][0] = x_1 \cdot cs - y_1 \cdot sn $$
        [S1, unofficial]
    *   Render the face using the projected coordinates and texture mapping [S2, unofficial].

**Parameters:**
*   **Surface Scale Multiplier:** A value of 120 is used for floor/ceiling projection calculations (e.g., `surfaceScale * 120` for tile calculation) [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point errors in camera position or block coordinate calculation may cause flickering if the camera is near block boundaries. Using consistent flooring and epsilon checks is recommended [G].
*   **Degenerate Input:** If the camera is exactly on a block boundary, the floor operation should consistently assign the block to one side to avoid ambiguity [G].
*   **Orientation Ambiguity:** When the view vector has equal components along multiple axes, the definition of `greatestDir` must be consistent between frames to prevent unnecessary mesh updates [S1, unofficial].

## Subtopics

### [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md)
Determines visibility of individual voxel faces based on normal-viewer angle. Choose when fine-grained control over face rendering is needed within a block, rather than skipping the entire block.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per block for the change detection step, as it involves simple integer comparisons and trigonometric precomputations [S1, unofficial]. Mesh regeneration is $O(F)$ where $F$ is the number of visible faces, but this is only triggered when the camera state changes [S1, unofficial].
*   **Memory Complexity:** $O(1)$ per block for storing visibility flags and previous camera state [S2, unofficial].
*   **Measured Performance:** In voxel engines like Exolands, this method significantly reduces CPU load by avoiding redundant mesh updates when the camera is stationary or moving slowly within the same block [S1, unofficial].
*   **Strengths:** Simple to implement, low computational overhead, effective for static or slow-moving cameras [S1, unofficial].
*   **Limitations:** Does not account for occlusion by other blocks or geometry; only considers camera position and orientation [S1, unofficial]. May still render blocks that are occluded by other geometry, requiring additional culling methods like [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) or [Mask Projection Culling](6836-mask-projection-culling.md).
*   **When to Use:** Ideal for voxel engines where camera movement is discrete or slow, and block-level updates are expensive.
*   **When Not to Use:** In scenes with fast camera movement or dynamic geometry where occlusion is significant, as it does not handle occlusion [S1, unofficial].

## Background
Block Visibility Culling is a technique used in voxel-based rendering engines to optimize performance by reducing the number of blocks that need to be processed and rendered. It is commonly found in indie voxel games and engines that use chunk-based mesh generation, such as Exolands and Voxel1993 [S1, unofficial][S2, unofficial]. The method leverages the fact that small camera movements within the same block do not change the visible geometry of that block, allowing the engine to skip expensive mesh regeneration steps.

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S2] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
