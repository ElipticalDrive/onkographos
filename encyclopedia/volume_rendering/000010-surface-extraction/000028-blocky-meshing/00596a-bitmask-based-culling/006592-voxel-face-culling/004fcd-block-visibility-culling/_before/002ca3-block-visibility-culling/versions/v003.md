# Block Visibility Culling

Also called: Block Coordinate Change Detection, Block Rendering & Surface Orientation Culling, Camera-relative Block Culling, renderBlock

## Overview
Block Visibility Culling is a method to determine which blocks need to be rendered based on camera position and rotation, avoiding unnecessary mesh updates when the camera has not moved significantly or changed its primary viewing direction [S1, unofficial]. It renders individual blocks by calculating screen coordinates relative to the player, determining surface orientation (top/bottom/front/back), and culling invisible faces based on player position and block visibility flags [S2, unofficial]. It sets itself apart from sibling methods by focusing on **camera-relative block-level culling** and **orientation-based face skipping** rather than just neighbor-based face culling (as in [Voxel Block Placement with Face Culling](923-voxel-block-placement-with-face-culling.md)) or global mesh projection (as in [Mesh Culling via 3D Projection and Connected Components](6963-mesh-culling-via-3d-projection-and-connected-components.md)).

## How it works
**Inputs:**
*   Camera position $(P_x, P_y, P_z)$ and rotation angle $\theta$ (in degrees, typically around the Y-axis for horizontal view).
*   Block grid data, where each block has a `visibleFaces` flag array.
*   Previous frame's camera block coordinates $(C_{x,prev}, C_{y,prev}, C_{z,prev})$ and greatest viewing direction `greatestDir`.

**Outputs:**
*   A list of blocks to render, with only their visible faces processed.

**Data Structures:**
*   `BlockFace` enum or flags: `YNeg`, `XPos`, `YPos`, `XNeg`, `ZNeg`, `ZPos`.
*   `block->visibleFaces[BlockFace::...]`: A boolean or bitmask indicating if a specific face is visible.
*   Camera state: `currentBlockX`, `currentBlockY`, `currentBlockZ`, `currentGreatestDir`.

**Steps:**

1.  **Camera Block Coordinate Calculation:**
    Compute the integer block coordinates of the camera by flooring its world position:
    $$ C_x = \lfloor P_x / \text{BLOCK\_SIZE} \rfloor $$
    $$ C_y = \lfloor P_y / \text{BLOCK\_SIZE} \rfloor $$
    $$ C_z = \lfloor P_z / \text{BLOCK\_SIZE} \rfloor $$

2.  **Change Detection (Optimization):**
    Compare the current camera block coordinates $(C_x, C_y, C_z)$ and the greatest viewing direction `greatestDir` with the previous frame's values. If both match, skip the index update process for this frame [S1, unofficial].

3.  **Block Iteration and Orientation:**
    For each block at chunk-relative coordinates $(x, y, z)$:
    *   Calculate world coordinates:
        $$ \text{blockX} = \text{chunkX} + x \times \text{BLOCK\_SIZE} $$
        $$ \text{blockY} = \text{chunkY} + y \times \text{BLOCK\_SIZE} $$
        $$ \text{blockZBottom} = \text{chunkZ} + z \times \text{BLOCK\_SIZE} $$
        $$ \text{blockZTop} = \text{blockZBottom} + \text{BLOCK\_SIZE} $$
    *   Determine surface orientation relative to the camera's Z-axis:
        $$ \text{surfaceOrientation} = \begin{cases} 1 & \text{if } P_z < \text{blockZBottom} \\ 2 & \text{if } P_z > \text{blockZTop} \\ 0 & \text{otherwise} \end{cases} $$
    *   Set rendering cycles:
        $$ \text{cycles} = \begin{cases} 2 & \text{if } \text{surfaceOrientation} \neq 0 \\ 1 & \text{otherwise} \end{cases} $$

4.  **Trigonometric Precomputation:**
    Calculate camera space rotation components:
    $$ c_s = \cos(\theta \times \pi / 180) $$
    $$ s_n = \sin(\theta \times \pi / 180) $$

5.  **View Vector Calculation:**
    Compute the view vector components $(v_x, v_y, v_z)$ from the camera's rotation angles $(r_x, r_y)$ [S1, unofficial].

6.  **Face Visibility Check and Rendering:**
    Loop through the block's faces (typically 4 side faces, indexed $w=0$ to $3$).
    *   For each face, check the corresponding `visibleFaces` flag:
        *   If $w=0$ and `!block->visibleFaces[BlockFace::YNeg]`, skip.
        *   If $w=1$ and `!block->visibleFaces[BlockFace::XPos]`, skip.
        *   If $w=2$ and `!block->visibleFaces[BlockFace::YPos]`, skip.
        *   If $w=3$ and `!block->visibleFaces[BlockFace::XNeg]`, skip [S2, unofficial].
    *   Transform block corners to camera space:
        $$ x_1 = x_{\text{Bottom1}}[w] - P_x $$
        $$ y_1 = y_{\text{Bottom1}}[w] - P_y $$
    *   Swap coordinates if orientation is 1 (ceiling):
        If $\text{surfaceOrientation} == 1$, swap $x_1/x_2$ and $y_1/y_2$.
    *   Project to screen space:
        $$ \text{wallX}[w][0] = x_1 \times c_s - y_1 \times s_n $$
    *   Render the face using the projected coordinates and texture mapping [S2, unofficial].

**Parameters:**
*   `surfaceScale`: Multiplier for floor/ceiling projection, typically 120 [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** When calculating distances or projections, avoid division by zero (e.g., if $z=0$ in perspective projection, use a small epsilon like 0.0001) [S2, unofficial].
*   **Orientation Logic:** The orientation check relies on the camera's Z-axis relative to the block's vertical extent. Incorrect axis alignment will cause faces to be culled incorrectly.
*   **State Caching:** The change detection optimization assumes that if the camera block and direction haven't changed, the visible block set hasn't changed. This may fail if the camera moves within the same block but changes its view of internal faces (though for block-level culling, this is often acceptable).

## Subtopics

### Voxel Face Visibility Check
Determines which of the six cube faces are visible by computing the dot product between each face's center and its outward normal, marking it visible if the dot product is positive [S3, unofficial].
*   **Steps:**
    1.  Compute `half_size = cube_size / 2`.
    2.  Define normals: Front(0,0,1), Back(0,0,-1), Right(-1,0,0), Left(1,0,0), Top(0,1,0), Bottom(0,-1,0).
    3.  Compute face centers: `cube_center + offset` for each face.
    4.  For each face, compute `dot_product = np.dot(face_center, normal)`.
    5.  Append `dot_product > 0` to visible list.
*   **Formula:** $v_i = \mathbf{c}_i \cdot \mathbf{n}_i > 0$ where $\mathbf{c}_i$ is the face center and $\mathbf{n}_i$ is the outward normal [S3, unofficial].
*   **Pitfall:** The visibility test uses the world origin (0,0,0) as the reference point for the dot product, not the camera position. This means faces are only marked visible if they point away from the world origin, which is incorrect for arbitrary camera positions [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of blocks in the view frustum. The change detection optimization reduces work to $O(1)$ if the camera state hasn't changed.
*   **Memory Complexity:** $O(1)$ per block for storing visibility flags.
*   **Performance:** Significantly reduces draw calls and vertex processing by skipping invisible faces and entire blocks when the camera is static [S1, unofficial].
*   **Strengths:** Simple to implement, effective for static or slowly moving cameras, low overhead.
*   **Limitations:** Does not handle occlusion from other blocks (only face orientation), requires frequent updates if the camera moves frequently.
*   **When to use:** Ideal for voxel engines with block-based rendering where camera movement is discrete or slow.
*   **When not to use:** Not suitable for dynamic scenes with fast-moving cameras or complex occlusion requirements.

## Background
Originating from early voxel rendering implementations, this method is exemplified by projects like `exolands` [S1, unofficial] and `Voxel1993` [S2, unofficial]. It builds on the principle of neighbor-based face culling but adds a layer of camera-relative optimization. The technique is common in indie voxel games and educational rendering engines due to its simplicity and effectiveness.

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S2] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
[S3] anb2473/Python-Voxel-Rendering-Engine, https://github.com/anb2473/Python-Voxel-Rendering-Engine, fetched 2026-10-27 (unofficial)
