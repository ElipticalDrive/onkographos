# Block Visibility Culling

Also called: Block Coordinate Change Detection, Block Rendering & Surface Orientation Culling, Camera-relative Block Culling, renderBlock

## Overview
Block Visibility Culling determines which voxel blocks require mesh updates or rendering based on the camera's position and rotation, avoiding unnecessary geometry generation when the viewing direction or position has not changed significantly [S1, unofficial]. It operates at the coarse block level by comparing current camera coordinates against previous frame values to skip index updates entirely, or by calculating screen-space projections to cull invisible faces based on surface orientation relative to the player [S2, unofficial]. Unlike [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md), which evaluates individual faces within a block, this method makes decisions about entire blocks or large face groups based on camera movement [S1, unofficial][S2, unofficial].

## How it works
**Inputs:** Camera world position $(P_x, P_y, P_z)$, camera rotation angle $\theta$ (in degrees), block grid coordinates $(x, y, z)$, block size $B$, chunk base coordinates $(C_x, C_y, C_z)$, and previous frame's camera block coordinates $(P'_x, P'_y, P'_z)$ and greatest viewing direction $D'$.

**Outputs:** Boolean flag indicating whether the block's mesh indices need updating, and a set of visible face flags for the block.

**Data Structures:**
*   `BlockFace`: An enumeration or bitmask with six flags: `YNeg` (bottom), `XPos` (right), `YPos` (top), `XNeg` (left), `ZNeg` (back), `ZPos` (front).
*   `visibleFaces`: An array or struct storing the visibility state for each of the six faces of a block.

**Steps:**

1.  **Compute Camera Block Coordinates:**
    Calculate the integer block coordinates containing the camera by flooring its world position:
    $$ P_{blockX} = \lfloor P_x \rfloor, \quad P_{blockY} = \lfloor P_y \rfloor, \quad P_{blockZ} = \lfloor P_z \rfloor $$
    [S1, unofficial]

2.  **Check for Stasis (Optimization):**
    Determine the current greatest viewing direction $D$ (e.g., based on yaw/pitch angles). If the current camera block coordinates $(P_{blockX}, P_{blockY}, P_{blockZ})$ match the previous frame's coordinates $(P'_x, P'_y, P'_z)$ AND the greatest viewing direction $D$ matches the previous direction $D'$, skip the index update process entirely. [S1, unofficial]

3.  **Calculate Block World Bounds:**
    For a block at grid coordinates $(x, y, z)$ within a chunk starting at $(C_x, C_y, C_z)$:
    $$ B_x = C_x + x \cdot B $$
    $$ B_y = C_y + y \cdot B $$
    $$ B_{zBottom} = C_z + z \cdot B $$
    $$ B_{zTop} = B_{zBottom} + B $$
    [S1, unofficial]

4.  **Determine Surface Orientation:**
    Evaluate the camera's vertical position relative to the block to determine the primary viewing orientation:
    $$ \text{orientation} = \begin{cases} 1 & \text{if } P_z < B_{zBottom} \quad (\text{looking up}) \\ 2 & \text{if } P_z > B_{zTop} \quad (\text{looking down}) \\ 0 & \text{otherwise} \quad (\text{looking straight}) \end{cases} $$
    [S1, unofficial][S2, unofficial]

5.  **Compute Trigonometric Constants:**
    Convert the camera's horizontal angle $\theta$ to radians and compute sine and cosine:
    $$ c_s = \cos\left(\theta \cdot \frac{\pi}{180}\right) $$
    $$ s_n = \sin\left(\theta \cdot \frac{\pi}{180}\right) $$
    [S1, unofficial]

6.  **Transform to Camera Space:**
    For each of the four vertical edges (walls) of the block, calculate the bottom corner coordinates relative to the camera. Let $(x_{bottom}, y_{bottom})$ be the 2D screen-space or world-space projection of the bottom corners.
    $$ x_1 = x_{bottom} - P_x $$
    $$ y_1 = y_{bottom} - P_y $$
    [S1, unofficial]

    Apply orientation-based swapping if looking up or down:
    If $\text{orientation} == 1$ (looking up):
    $$ \text{swap}(x_1, x_2) $$
    $$ \text{swap}(y_1, y_2) $$
    [S1, unofficial]

7.  **Project to Screen/View Coordinates:**
    Compute the horizontal screen coordinates for the wall vertices:
    $$ \text{wallX}[w][0] = x_1 \cdot c_s - y_1 \cdot s_n $$
    [S1, unofficial]
    *(Note: Full projection to screen space involves further scaling and offsetting, but this step establishes the relative horizontal position for culling.)*

8.  **Cull Invisible Faces:**
    Before rendering or adding indices for a specific face $w$ (where $w \in \{0, 1, 2, 3\}$ corresponds to YNeg, XPos, YPos, XNeg respectively), check the `visibleFaces` flag.
    $$ \text{if } w == 0 \text{ and } \neg \text{visibleFaces}[\text{YNeg}] \text{ then continue} $$
    $$ \text{if } w == 1 \text{ and } \neg \text{visibleFaces}[\text{XPos}] \text{ then continue} $$
    $$ \text{if } w == 2 \text{ and } \neg \text{visibleFaces}[\text{YPos}] \text{ then continue} $$
    $$ \text{if } w == 3 \text{ and } \neg \text{visibleFaces}[\text{XNeg}] \text{ then continue} $$
    [S2, unofficial]

    The `visibleFaces` flags are typically determined by the geometric relationship between the face normal and the camera view vector, or by the `orientation` calculated in Step 4. For example, if looking up (orientation 1), the bottom face (YNeg) might be hidden, or if looking down (orientation 2), the top face (YPos) might be hidden.

**Parameters:**
*   `surfaceScale`: A multiplier for floor/ceiling projection calculations, typically 120. Used as `surfaceScale * 120` for tile calculation. [S2, unofficial]

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point comparisons for camera position should use epsilon or integer block coordinates to avoid flickering when the camera is on a block boundary. [S1, unofficial]
*   **Degenerate Input:** If the block width or height in screen space is zero (e.g., $xPos2 - xPos1 == 0$), division by zero must be avoided in projection steps. [S2, unofficial]
*   **Orientation Ambiguity:** When the camera is exactly at the block's vertical center, orientation is 0, requiring careful handling of which faces are considered "front" or "back" relative to the view vector. [S1, unofficial]

## Subtopics

### [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md)
Determines visibility of individual faces within a block based on geometric normals. Operates at a finer granularity than block-level culling.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per block for the stasis check (Step 2). $O(N)$ for blocks requiring update, where $N$ is the number of faces. The culling reduces the constant factor significantly by skipping geometry generation for hidden faces.
*   **Memory Complexity:** $O(1)$ additional memory per block to store `visibleFaces` and previous camera state.
*   **Performance:** Significantly reduces draw calls and vertex buffer updates when the camera is stationary or moving slowly within a block. [S1, unofficial]
*   **Strengths:** Simple to implement; highly effective for static scenes or slow-moving cameras; reduces CPU load during mesh generation.
*   **Limitations:** Does not cull faces based on occlusion by other blocks (requires [Face Culling](8735-face-culling.md) or [Opacity-based Mesh Culling](8759-opacity-based-mesh-culling.md)); less effective for fast-moving cameras where the stasis check rarely triggers.

## Background
Block Visibility Culling is a technique commonly used in voxel-based games and renderers to optimize rendering performance. It is implemented in projects such as [exolands](https://github.com/kvxy/exolands) [S1, unofficial] and [Voxel1993](https://github.com/VidFurlan/Voxel1993) [S2, unofficial]. The method leverages the fact that many voxel blocks are only partially visible from a given camera angle, and that small camera movements often do not change the set of visible blocks.

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S2] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
