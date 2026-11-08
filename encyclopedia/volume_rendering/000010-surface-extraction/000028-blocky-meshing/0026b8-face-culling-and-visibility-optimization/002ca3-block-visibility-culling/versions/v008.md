# Block Visibility Culling

Also called: Block Coordinate Change Detection, Block Rendering & Surface Orientation Culling, Camera-relative Block Culling, renderBlock

## Overview
Block Visibility Culling is a coarse-grained optimization technique that determines which voxel blocks require mesh updates or rendering based on the camera's position and orientation, avoiding unnecessary geometry generation when the viewing direction has not changed significantly [S1, unofficial]. Unlike sibling methods that operate at the fine-grained face level within a block (e.g., [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md)), this method operates at the block level, using camera-relative coordinates to cull entire faces of a block before they are processed [S2, unofficial]. It serves as a pre-filter to reduce the computational load of subsequent face-level culling and mesh assembly.

## How it works
**Inputs and Outputs:**
*   **Input:** Camera world position $(P_x, P_y, P_z)$, camera rotation angle $\theta$ (in degrees), block grid coordinates $(x, y, z)$, block size $B$, and previous frame's camera block coordinates and "greatest direction" [S1, unofficial].
*   **Output:** A set of visible faces for the block, or a flag indicating the block can be skipped entirely [S1, unofficial].

**Data Structures:**
*   **Block Face Flags:** An enumeration or bitmask `BlockFace` with six flags: `YNeg`, `XPos`, `YPos`, `XNeg`, `ZNeg`, `ZPos` [S2, unofficial].
*   **Camera State:** Stores integer block coordinates of the camera and the dominant viewing axis (greatestDir) from the previous frame [S1, unofficial].

**Steps:**
1.  **Compute Camera Block Coordinates:**
    Calculate the integer block coordinates containing the camera by flooring its world position:
    $$ B_{cam} = \lfloor P_{cam} / B \rfloor $$
    [S1, unofficial]

2.  **Check for Stasis (Optimization):**
    Compare the current camera block coordinates and `greatestDir` with the values from the previous frame. If both match, skip the index update process for all blocks in the current view frustum [S1, unofficial].

3.  **Calculate Camera Space Rotation:**
    Compute the sine and cosine of the camera's horizontal rotation angle $\theta$:
    $$ c_s = \cos(\theta \cdot \pi / 180) $$
    $$ s_n = \sin(\theta \cdot \pi / 180) $$
    [S2, unofficial]

4.  **Determine Surface Orientation:**
    For a block at grid position $(x, y, z)$ with size $B$:
    *   Calculate the block's vertical bounds in world space:
        $$ Z_{bottom} = z \cdot B $$
        $$ Z_{top} = (z + 1) \cdot B $$
    *   Determine the vertical orientation relative to the camera's height $P_z$:
        $$ O_z = \begin{cases} 1 & \text{if } P_z < Z_{bottom} \quad (\text{Camera below block}) \\ 2 & \text{if } P_z > Z_{top} \quad (\text{Camera above block}) \\ 0 & \text{otherwise} \quad (\text{Camera inside block height}) \end{cases} $$
    [S2, unofficial]

5.  **Compute View Vector Components:**
    Calculate the view vector components $(v_x, v_y, v_z)$ from the camera's rotation angles (typically horizontal $\theta$ and vertical pitch $\phi$) to determine the primary viewing direction [S1, unofficial].

6.  **Transform Block Corners to Camera Space:**
    For each of the four bottom corners $w \in \{0, 1, 2, 3\}$ of the block:
    *   Calculate world coordinates of the corner:
        $$ X_{world} = x \cdot B + w_x \cdot B $$
        $$ Y_{world} = y \cdot B + w_y \cdot B $$
        $$ Z_{world} = z \cdot B $$
    *   Translate to camera space:
        $$ x_1 = X_{world} - P_x $$
        $$ y_1 = Y_{world} - P_y $$
    *   Rotate by camera angle:
        $$ wallX[w][0] = x_1 \cdot c_s - y_1 \cdot s_n $$
    [S2, unofficial]

7.  **Cull Invisible Faces:**
    Iterate through the four side faces ($w=0..3$). For each face, check the corresponding `BlockFace` flag. If the flag is not set, skip rendering that face [S2, unofficial].
    *   Note: The `visibleFaces` array is typically populated by checking neighbor occupancy (as per [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md)) and potentially by the orientation logic above [S2, unofficial].

**Parameters:**
*   **Surface Scale Multiplier:** A constant value of 120 is used for floor/ceiling projection calculations in some implementations [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** When calculating distances or projections, avoid division by zero. If a denominator (e.g., $z$ in perspective projection) is zero, use a small epsilon value (e.g., 0.0001) [S2, unofficial].
*   **Boundary Conditions:** Ensure that block coordinates are correctly clamped or handled when the camera is exactly on a block boundary [S1, unofficial].

## Subtopics

### [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md)
Determines visibility of individual faces within a block based on geometric relationships with the camera. Use this when fine-grained control over face rendering is needed, as opposed to the coarse block-level culling of the parent method.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per block for the culling decision, assuming constant-time coordinate checks. The stasis check allows skipping entire batches of blocks, reducing effective complexity to $O(N_{changed})$ where $N_{changed}$ is the number of blocks affected by camera movement.
*   **Memory Complexity:** $O(1)$ auxiliary memory per block for storing face flags and camera state.
*   **Strengths:** Significantly reduces the number of triangles submitted to the graphics pipeline by avoiding mesh updates for static views. Simple to implement and integrate with existing voxel engines.
*   **Limitations:** Only effective when the camera is stationary or moving slowly. Does not account for occlusion by other blocks (that is handled by neighbor checks in the parent method).
*   **When to use:** Ideal for voxel-based games or applications where the camera moves discretely or pauses frequently, allowing for batch updates.
*   **When not to use:** In scenarios with rapid, continuous camera movement where the overhead of checking stasis might outweigh the benefits, or when fine-grained face culling is insufficient and full mesh reconstruction is required.

## Background
Block Visibility Culling originated in early voxel-based rendering engines, particularly those inspired by games like Minecraft. It was implemented in various open-source projects to optimize rendering performance by reducing the frequency of mesh updates. The method is documented in unofficial repositories such as `kvxy/exolands` and `VidFurlan/Voxel1993`, which provide practical implementations for educational and hobbyist purposes [S1, unofficial][S2, unofficial].

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S2] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
