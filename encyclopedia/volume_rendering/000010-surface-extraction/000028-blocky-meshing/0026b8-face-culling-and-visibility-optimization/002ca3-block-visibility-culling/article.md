# Block Visibility Culling

Also called: Block Coordinate Change Detection, Block Rendering & Surface Orientation Culling, Camera-relative Block Culling, renderBlock

## Overview
Block Visibility Culling determines which voxel blocks require mesh updates or rendering based on the camera's position and rotation, avoiding unnecessary geometry generation when the camera has not moved significantly or changed its primary viewing direction [S1, unofficial]. Unlike sibling methods that operate at the fine-grained face level within a block [6723-voxel-face-visibility-check.md], this method operates at the coarse block level, using camera-relative coordinates to cull entire blocks or specific faces based on surface orientation relative to the player [S2, unofficial]. It builds on [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md) by adding camera-relative visibility checks.

## How it works
**Inputs:**
*   Camera position $(P_x, P_y, P_z)$ and rotation angle $\theta$ (in degrees).
*   Block dimensions: `BLOCK_SIZE` (typically 16 in [S2, unofficial]).
*   Block grid coordinates $(chunkX, chunkY, chunkZ)$ and local block coordinates $(x, y, z)$.
*   Previous frame's camera block coordinates and greatest viewing direction.
*   Block face visibility flags: `BlockFace::YNeg`, `XPos`, `YPos`, `XNeg`, `ZNeg`, `ZPos` [S2, unofficial].

**Outputs:**
*   A set of visible faces for the block, or a decision to skip rendering/updating the block entirely.

**Data Structures:**
*   `BlockFace` enum or flags array storing visibility for the six faces [S2, unofficial].
*   `surfaceOrientation`: Integer flag (0, 1, or 2) indicating the block's vertical relationship to the camera [S2, unofficial].

**Steps:**

1.  **Compute Camera Block Coordinates:**
    Calculate the integer block coordinates containing the camera by flooring its world position:
    $$
    P_{cam\_block} = \lfloor P_{camera} / \text{BLOCK\_SIZE} \rfloor
    $$
    [S1, unofficial]

2.  **Check for Redundant Updates:**
    Compare the current camera block coordinates and the greatest viewing direction (derived from camera rotation) with the previous frame's values. If both match, skip the index update process for this block [S1, unofficial].

3.  **Determine Surface Orientation:**
    Calculate the block's vertical bounds relative to the camera:
    $$
    \begin{aligned}
    \text{blockZBottom} &= \text{chunkZ} + z \times \text{BLOCK\_SIZE} \\
    \text{blockZTop} &= \text{blockZBottom} + \text{BLOCK\_SIZE}
    \end{aligned}
    $$
    Determine `surfaceOrientation`:
    *   If $P_z < \text{blockZBottom}$, `surfaceOrientation` = 1 (camera is below the block) [S2, unofficial].
    *   If $P_z > \text{blockZTop}$, `surfaceOrientation` = 2 (camera is above the block) [S2, unofficial].
    *   Otherwise, `surfaceOrientation` = 0 (camera is within the block's vertical range) [S2, unofficial].

4.  **Calculate Cycles:**
    Set the number of rendering cycles `cycles`:
    *   If `surfaceOrientation` $\neq 0$, `cycles` = 2 [S2, unofficial].
    *   Else, `cycles` = 1 [S2, unofficial].

5.  **Compute Camera Space Coordinates:**
    Calculate the camera space rotation components:
    $$
    \begin{aligned}
    cs &= \cos(\theta \times \pi / 180) \\
    sn &= \sin(\theta \times \pi / 180)
    \end{aligned}
    $$
    [S2, unofficial]
    For each corner $w$ (0 to 3) of the block's bottom face, compute the bottom corner positions in world space, then transform to camera space:
    $$
    \begin{aligned}
    x1 &= x_{\text{Bottom1}}[w] - P_x \\
    y1 &= y_{\text{Bottom1}}[w] - P_y
    \end{aligned}
    $$
    [S2, unofficial]
    If `surfaceOrientation` == 1, swap $x1/x2$ and $y1/y2$ [S2, unofficial].
    Project to screen space:
    $$
    \text{wallX}[w][0] = x1 \times cs - y1 \times sn
    $$
    [S2, unofficial]

6.  **Face Visibility Check:**
    Before rendering each face $w$ (0 to 3 corresponding to YNeg, XPos, YPos, XNeg), check the block's visibility flags:
    *   If $w=0$ and `!block->visibleFaces[BlockFace::YNeg]`, skip [S2, unofficial].
    *   If $w=1$ and `!block->visibleFaces[BlockFace::XPos]`, skip [S2, unofficial].
    *   If $w=2$ and `!block->visibleFaces[BlockFace::YPos]`, skip [S2, unofficial].
    *   If $w=3$ and `!block->visibleFaces[BlockFace::XNeg]`, skip [S2, unofficial].
    [S2, unofficial]

**Parameters:**
*   `surfaceScale`: Multiplier for floor/ceiling projection, typically 120 [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** When calculating distances or projections, avoid division by zero (e.g., if $z=0$ in perspective projection, use a small epsilon like 0.0001) [S2, unofficial].
*   **Boundary Conditions:** Ensure `surfaceOrientation` correctly handles cases where the camera is exactly on the boundary of a block's vertical range [S2, unofficial].

## Subtopics

### [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md)
Determines visibility of individual faces within a block based on geometric relationships with the viewer. Choose this when fine-grained face culling is needed within a block, as opposed to block-level culling.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per block for the visibility check, assuming constant-time coordinate and orientation calculations. The check is skipped entirely if camera position and direction haven't changed [S1, unofficial].
*   **Memory Complexity:** $O(1)$ per block for storing visibility flags and orientation state [S2, unofficial].
*   **Strengths:** Significantly reduces unnecessary mesh updates and rendering calls when the camera is stationary or moving slowly [S1, unofficial]. Simple to implement using basic trigonometry and coordinate comparisons [S2, unofficial].
*   **Limitations:** Only effective for camera-relative culling; does not handle occlusion by other blocks (that is handled by [Face Culling](8735-face-culling.md) or [Voxel Backface Culling](4569-voxel-backface-culling.md)). Requires tracking previous frame state [S1, unofficial].
*   **When to Use:** Ideal for voxel engines where camera movement is discrete or slow, and block-level updates are expensive.
*   **When Not to Use:** When the camera moves rapidly or continuously, as the overhead of checking state changes may not be justified if most blocks still need updating. Also ineffective for static scenes where occlusion is the primary concern.

## Background
Block Visibility Culling is a technique used in voxel-based rendering engines to optimize performance by avoiding redundant calculations for blocks that are not visible due to camera position or orientation. It is implemented in projects like `exolands` [S1, unofficial] and `Voxel1993` [S2, unofficial]. The method leverages the fact that many blocks remain in the same relative position to the camera, allowing the engine to skip updates when the camera's block coordinates and viewing direction remain unchanged [S1, unofficial].

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S2] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
