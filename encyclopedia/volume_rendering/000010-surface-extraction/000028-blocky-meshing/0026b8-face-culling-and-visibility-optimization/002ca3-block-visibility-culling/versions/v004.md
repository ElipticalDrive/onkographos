# Block Visibility Culling

Also called: Block Coordinate Change Detection, Block Rendering & Surface Orientation Culling, Camera-relative Block Culling, renderBlock

## Overview
Block Visibility Culling determines which voxel blocks require mesh updates or rendering based on the camera's position and rotation, avoiding unnecessary geometry processing when the camera has not moved significantly or changed its primary viewing direction [S1, unofficial][S2, unofficial]. Unlike its parent [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md), which operates on individual voxel faces, this method operates at the coarse block level, using camera-relative orientation to cull entire block faces before they are processed by the renderer [S2, unofficial]. It differs from [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md) by evaluating visibility at the block coordinate level rather than the fine-grained face level within a block [S2, unofficial].

## How it works
**Inputs:** Camera world position $(P_x, P_y, P_z)$, camera rotation angle $\theta$ (in degrees), block size $B$, chunk origin $(C_x, C_y, C_z)$, and previous frame's camera block coordinates and greatest direction.
**Outputs:** A set of visible block faces to render, or a skip flag if no update is needed.

**Data Structures:**
*   `BlockFace`: An enum or bitmask with flags `YNeg`, `XPos`, `YPos`, `XNeg`, `ZNeg`, `ZPos` representing the six faces of a block [S2, unofficial].
*   `visibleFaces`: An array or bitmask stored per block, indicating which faces are visible [S2, unofficial].

**Steps:**
1.  **Camera Block Coordinate Calculation:** Compute the integer block coordinates of the camera by flooring its world position:
    $$ B_{cam} = \lfloor P / B \rfloor $$
    [S1, unofficial]

2.  **Change Detection:** Compare the current camera block coordinates and the greatest direction of the view vector with the previous frame's values. If both match, skip the index update process entirely [S1, unofficial].

3.  **View Vector Calculation:** Compute the view vector components $(v_x, v_y, v_z)$ from the camera's rotation angles $(r_x, r_y)$ [S1, unofficial].

4.  **Surface Orientation Determination:** For a block at local coordinates $(x, y, z)$ within a chunk:
    *   Calculate world coordinates:
        $$ B_x = C_x + x \cdot B $$
        $$ B_y = C_y + y \cdot B $$
        $$ B_{z, bottom} = C_z + z \cdot B $$
        $$ B_{z, top} = B_{z, bottom} + B $$
        [S1, unofficial]
    *   Determine surface orientation relative to the camera's vertical position $P_z$:
        $$ \text{orientation} = \begin{cases} 1 & \text{if } P_z < B_{z, bottom} \\ 2 & \text{if } P_z > B_{z, top} \\ 0 & \text{otherwise} \end{cases} $$
        [S1, unofficial]

5.  **Cycle Count:** Set the number of rendering cycles $N_{cycles}$:
    $$ N_{cycles} = \begin{cases} 2 & \text{if orientation} \neq 0 \\ 1 & \text{otherwise} \end{cases} $$
    [S1, unofficial]

6.  **Face Visibility Check:** For each of the four side faces (indexed $w \in [0, 3]$), check the corresponding `visibleFaces` flag before rendering:
    *   $w=0$: Check `BlockFace::YNeg`
    *   $w=1$: Check `BlockFace::XPos`
    *   $w=2$: Check `BlockFace::YPos`
    *   $w=3$: Check `BlockFace::XNeg`
    If the flag is not set, skip rendering that face [S2, unofficial].

7.  **Projection and Rendering:**
    *   Compute trigonometric values: $c_s = \cos(\theta \cdot \pi / 180)$, $s_n = \sin(\theta \cdot \pi / 180)$ [S1, unofficial].
    *   Transform block corners to camera space:
        $$ x_1 = x_{bottom1}[w] - P_x $$
        $$ y_1 = y_{bottom1}[w] - P_y $$
        [S1, unofficial]
    *   Swap $x_1/x_2$ and $y_1/y_2$ if $\text{orientation} == 1$ [S1, unofficial].
    *   Project to screen space:
        $$ \text{wallX}[w][0] = x_1 \cdot c_s - y_1 \cdot s_n $$
        [S1, unofficial]
    *   Render the face using the calculated screen coordinates and texture mapping [S2, unofficial].

**Parameters:**
*   `surfaceScale`: Multiplier for floor/ceiling projection, typically 120 [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point comparisons for camera position should use a small epsilon to avoid flickering when the camera is exactly on a block boundary [G].
*   **Degenerate Input:** If the block width or height in screen space is zero, avoid division by zero during texture coordinate calculation [S2, unofficial].

## Subtopics

### [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md)
Determines visibility of individual faces within a block based on geometric relationship with the viewer. Choose this when fine-grained culling within a block is needed, rather than block-level culling.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per block for change detection; $O(F)$ for rendering visible faces, where $F$ is the number of visible faces.
*   **Memory Complexity:** $O(1)$ per block for storing visibility flags.
*   **Performance:** Significantly reduces draw calls by skipping blocks that have not changed relative to the camera [S1, unofficial].
*   **Strengths:** Simple to implement, effective for static scenes or slow-moving cameras.
*   **Limitations:** Does not account for occlusion by other blocks, only camera-relative orientation [S2, unofficial].
*   **When to use:** Ideal for voxel engines where camera movement is discrete or slow, and block-level updates are expensive.
*   **When not to use:** In fast-moving scenes or dynamic environments where block visibility changes frequently.

## Background
Originating from early voxel engine implementations, this method was popularized by projects like Exolands and Voxel1993, which optimized rendering by reducing unnecessary mesh updates based on camera position [S1, unofficial][S2, unofficial]. It is commonly used in Minecraft-like games and voxel-based simulations.

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S2] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
