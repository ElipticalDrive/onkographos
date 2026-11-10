# Block Visibility Culling

Also called: Block Coordinate Change Detection, Block Rendering & Surface Orientation Culling, Camera-relative Block Culling, renderBlock

## Overview
Block Visibility Culling determines which voxel blocks require mesh updates or rendering based on the camera's position and rotation, avoiding unnecessary geometry generation when the viewing direction has not changed significantly [S1, S2, unofficial]. Unlike sibling methods that operate at the fine-grained face level within a block [6723-voxel-face-visibility-check.md], this method operates at the coarse block level, using camera-relative coordinates to cull entire blocks or faces based on surface orientation and visibility flags [S2, unofficial]. It builds on [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md) by adding a camera-centric pre-check to skip expensive per-face calculations when the block's visibility state is unchanged.

## How it works
**Inputs:** Camera world position $(P_x, P_y, P_z)$, camera rotation angle $\theta$ (in degrees), block grid coordinates $(x, y, z)$, block size $B$, chunk base coordinates $(C_x, C_y, C_z)$, and block visibility flags $V_{face}$ for each of the six faces.
**Outputs:** A set of visible faces for the block, or a flag indicating the block can be skipped entirely.

**Data Structures:**
*   `BlockFace`: An enum or bitfield with flags `YNeg`, `XPos`, `YPos`, `XNeg`, `ZNeg`, `ZPos` [S2, unofficial].
*   `visibleFaces`: An array or struct storing the current visibility state of each face [S2, unofficial].
*   `previousCameraBlock`: Stores the integer block coordinates of the camera from the previous frame [S1, unofficial].
*   `previousGreatestDir`: Stores the primary viewing direction from the previous frame [S1, unofficial].

**Steps:**
1.  **Compute Camera Block Coordinates:**
    Calculate the integer block coordinates containing the camera:
    $$ B_{cam} = \lfloor P / B \rfloor $$
    where the division is component-wise [S1, unofficial].

2.  **Check for Stasis (Optimization):**
    Compare $B_{cam}$ and the current greatest viewing direction with `previousCameraBlock` and `previousGreatestDir`. If both match, skip the index update process for this frame [S1, unofficial]. [G] The greatest viewing direction can be derived from the camera's forward vector components $(v_x, v_y, v_z)$ calculated from rotation angles [S1, unofficial].

3.  **Calculate Block World Coordinates:**
    For a block at grid position $(x, y, z)$ within a chunk starting at $(C_x, C_y, C_z)$:
    $$ X_{block} = C_x + x \cdot B $$
    $$ Y_{block} = C_y + y \cdot B $$
    $$ Z_{blockBottom} = C_z + z \cdot B $$
    $$ Z_{blockTop} = Z_{blockBottom} + B $$
    [S2, unofficial]

4.  **Determine Surface Orientation:**
    Evaluate the camera's vertical position relative to the block:
    $$
    \text{orientation} =
    \begin{cases}
    1 & \text{if } P_z < Z_{blockBottom} \quad (\text{Looking Up}) \\
    2 & \text{if } P_z > Z_{blockTop} \quad (\text{Looking Down}) \\
    0 & \text{otherwise} \quad (\text{Level})
    \end{cases}
    $$
    [S2, unofficial]

5.  **Compute Trigonometric Constants:**
    $$ c_s = \cos(\theta \cdot \pi / 180) $$
    $$ s_n = \sin(\theta \cdot \pi / 180) $$
    [S2, unofficial]

6.  **Transform to Camera Space:**
    For each corner of the block's bottom face (indexed $w=0..3$):
    $$ x_{bottom}[w] = X_{block} + \text{cornerOffset}_x[w] $$
    $$ y_{bottom}[w] = Y_{block} + \text{cornerOffset}_y[w] $$
    $$ x_1 = x_{bottom}[w] - P_x $$
    $$ y_1 = y_{bottom}[w] - P_y $$
    [S2, unofficial]

7.  **Apply Orientation Swap:**
    If $\text{orientation} == 1$ (Looking Up), swap $x_1$ and $y_1$ [S2, unofficial].

8.  **Project to Screen Space (Partial):**
    $$ \text{wallX}[w][0] = x_1 \cdot c_s - y_1 \cdot s_n $$
    [S2, unofficial]
    [G] Full projection requires further transformation to screen coordinates, but this step establishes the relative horizontal position for culling decisions.

9.  **Face Visibility Check:**
    For each face $f \in \{\text{YNeg, XPos, YPos, XNeg, ZNeg, ZPos}\}$:
    $$ \text{Visible}_f = V_{face}[f] \quad \text{AND} \quad \text{FaceNotCulledByOrientation}(f, \text{orientation}) $$
    [S2, unofficial]
    [G] `FaceNotCulledByOrientation` returns false if the face is back-facing relative to the camera's primary axis, based on the orientation value. For example, if looking up (orientation 1), bottom faces (YNeg) might be culled if not explicitly flagged, or top faces (YPos) prioritized. The specific logic depends on the implementation's handling of `surfaceOrientation` [S2, unofficial].

10. **Update State:**
    Store $B_{cam}$ and current viewing direction in `previousCameraBlock` and `previousGreatestDir` for the next frame [S1, unofficial].

**Parameters:**
*   `surfaceScale`: A multiplier for floor/ceiling projection, typically 120 [S2, unofficial].
*   `BLOCK_SIZE`: The size of a voxel block, typically 16 units in some implementations [S2, unofficial].

**Edge Cases:**
*   **Camera on Block Boundary:** Flooring handles this, but precision errors near integer boundaries may cause flickering. [G] Use a small epsilon when comparing camera position to block boundaries.
*   **Zero Division:** In projection formulas, ensure denominators (like $z$ in perspective division) are not zero. [G] Clamp $z$ to a minimum value (e.g., 0.0001) [S2, unofficial].
*   **Degenerate Input:** If the block is completely occluded by other blocks, the visibility flags should already reflect this, and the culling step will skip it.

## Subtopics

### [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md)
Determines visibility of individual faces within a block based on geometric normals and camera direction. Choose this when fine-grained control over face rendering is needed, unlike the coarse block-level culling of the parent method.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per block for the culling check, as it involves simple comparisons and trigonometric calculations [S1, S2, unofficial].
*   **Memory Complexity:** $O(1)$ per block for storing visibility flags and previous state [S2, unofficial].
*   **Performance:** Significantly reduces mesh update overhead when the camera is stationary or moving slowly, as the stasis check skips index updates entirely [S1, unofficial].
*   **Strengths:** Simple to implement, low computational cost, effective for static or slow-moving cameras [S1, S2, unofficial].
*   **Limitations:** Does not account for occlusion by other blocks (requires additional culling methods like [Mask Projection Culling](6836-mask-projection-culling.md) or [Mesh Culling via 3D Projection and Connected Components](6963-mesh-culling-via-3d-projection-and-connected-components.md)). Relies on accurate camera position tracking [S1, unofficial].
*   **When to Use:** Ideal for real-time applications where camera movement is the primary driver of visibility changes, and block-level granularity is sufficient.
*   **When Not to Use:** In scenes with fast-moving cameras or complex occlusion where per-face or per-voxel culling is necessary.

## Background
Block Visibility Culling originated in early voxel-based game engines, particularly those inspired by Minecraft-like aesthetics. It was implemented in projects like `exolands` [S1, unofficial] and `Voxel1993` [S2, unofficial] to optimize rendering by avoiding unnecessary geometry generation. The method leverages the fact that many blocks remain visible or invisible between frames if the camera does not move significantly, allowing for a simple state comparison to skip expensive mesh updates.

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S2] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
