# Block Visibility Culling

Also called: Block Coordinate Change Detection, Block Rendering & Surface Orientation Culling, Camera-relative Block Culling, renderBlock

## Overview
Block Visibility Culling determines which voxel blocks require mesh updates or rendering based on the camera's position and rotation, avoiding unnecessary geometry generation when the view has not changed significantly [S1, S2, unofficial]. It operates at the coarse block level by comparing current camera coordinates and viewing direction against previous frame values, skipping index updates if both match [S1, unofficial]. Unlike sibling methods that cull individual faces based on neighbor occupancy or surface orientation, this method optimizes the entire block's visibility state relative to camera movement [S1, S2, unofficial].

## How it works
**Inputs:** Camera world position $(C_x, C_y, C_z)$, camera rotation angles (yaw $A$, pitch $P$), previous frame's camera block coordinates $(B_{x,prev}, B_{y,prev}, B_{z,prev})$ and greatest viewing direction $D_{prev}$, block size $S$, chunk offsets $(ChunkX, ChunkY, ChunkZ)$.
**Outputs:** Boolean flag indicating if block indices need updating; if true, the set of visible faces for the block.

**Data Structures:**
*   `BlockFace`: An enum or bitmask with flags `YNeg`, `XPos`, `YPos`, `XNeg`, `ZNeg`, `ZPos` representing the six faces of a block [S2, unofficial].
*   `visibleFaces`: An array or bitmask storing which faces of the current block are visible [S2, unofficial].

**Steps:**
1.  **Compute Camera Block Coordinates:**
    Calculate the integer block coordinates containing the camera by flooring the world position:
    $$ B_x = \lfloor C_x / S \rfloor, \quad B_y = \lfloor C_y / S \rfloor, \quad B_z = \lfloor C_z / S \rfloor $$
    [S1, unofficial]

2.  **Determine Viewing Direction:**
    Compute the view vector components $(V_x, V_y, V_z)$ from the camera's rotation angles (rx, ry) [S1, unofficial]. Determine the "greatest direction" $D$ (the axis with the largest absolute component in the view vector) [S1, unofficial].

3.  **Change Detection:**
    Compare the current block coordinates $(B_x, B_y, B_z)$ and greatest direction $D$ with the previous frame's values $(B_{x,prev}, B_{y,prev}, B_{z,prev})$ and $D_{prev}$ [S1, unofficial].
    *   If $B_x = B_{x,prev}$ AND $B_y = B_{y,prev}$ AND $B_z = B_{z,prev}$ AND $D = D_{prev}$, skip the index update process entirely [S1, unofficial].
    *   Otherwise, proceed to face culling.

4.  **Surface Orientation and Face Culling:**
    For each block in the view frustum (or specifically the block containing the camera):
    a.  Calculate block boundaries:
        $$ BlockX = ChunkX + x \cdot S, \quad BlockY = ChunkY + y \cdot S $$
        $$ BlockZ_{Bottom} = ChunkZ + z \cdot S, \quad BlockZ_{Top} = BlockZ_{Bottom} + S $$
        [S2, unofficial]
    b.  Determine surface orientation relative to the player/camera:
        $$ Orientation = \begin{cases} 1 & \text{if } C_z < BlockZ_{Bottom} \\ 2 & \text{if } C_z > BlockZ_{Top} \\ 0 & \text{otherwise} \end{cases} $$
        [S2, unofficial]
    c.  Set cycle count: $Cycles = 2$ if $Orientation \neq 0$, else $1$ [S2, unofficial].
    d.  Transform block corners to camera space:
        $$ x_1 = x_{Bottom1}[w] - C_x, \quad y_1 = y_{Bottom1}[w] - C_y $$
        [S2, unofficial]
    e.  Swap $x_1/x_2$ and $y_1/y_2$ if $Orientation == 1$ [S2, unofficial].
    f.  Project to screen space:
        $$ cs = \cos(A \cdot \pi / 180), \quad sn = \sin(A \cdot \pi / 180) $$
        $$ WallX[w][0] = x_1 \cdot cs - y_1 \cdot sn $$
        [S2, unofficial]
    g.  Check visibility flags for each face $w \in \{0, 1, 2, 3\}$ (representing YNeg, XPos, YPos, XNeg):
        *   If $w=0$ and `!block->visibleFaces[BlockFace::YNeg]`, skip [S2, unofficial].
        *   If $w=1$ and `!block->visibleFaces[BlockFace::XPos]`, skip [S2, unofficial].
        *   If $w=2$ and `!block->visibleFaces[BlockFace::YPos]`, skip [S2, unofficial].
        *   If $w=3$ and `!block->visibleFaces[BlockFace::XNeg]`, skip [S2, unofficial].
        [S2, unofficial]

**Parameters:**
*   `surfaceScale`: Multiplier for floor/ceiling projection, typically 120 [S2, unofficial].
*   `BLOCK_SIZE`: Standard block dimension, often 16 units [S2, unofficial].

**Edge Cases:**
*   **Precision:** Floating point comparisons for camera position should use a small epsilon to avoid flickering when the camera is near block boundaries [G].
*   **Out of Bounds:** Neighbor checks must handle coordinates outside the world grid [S1, unofficial].

## Subtopics

### [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md)
Determines visibility of individual faces within a block based on geometric relationship to the viewer. Operates at a finer granularity than Block Visibility Culling, which works at the block level based on camera movement. Choose this when fine-grained face culling is needed within a block already determined to be visible.

## Operating characteristics
*   **Time Complexity:** $O(1)$ for change detection per block if coordinates match; $O(N)$ for face culling if change detected, where $N$ is the number of faces per block (constant 6).
*   **Memory Complexity:** $O(1)$ per block to store visibility flags and previous state.
*   **Strengths:** Significantly reduces unnecessary mesh updates and rendering calls when the camera is stationary or moving slowly within the same block [S1, unofficial]. Simple to implement [S1, S2, unofficial].
*   **Limitations:** Does not cull faces based on occlusion or neighbor occupancy (handled by parent [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md)). Only effective when camera movement is the primary driver of visibility changes [S1, unofficial].
*   **When to Use:** Real-time voxel engines where camera movement is frequent but often incremental, and mesh generation is expensive.
*   **When Not to Use:** Static scenes or when occlusion culling is more critical than camera-relative updates.

## Background
Originating from early voxel rendering implementations in game engines, this method was popularized by projects like Exolands [S1, unofficial] and Voxel1993 [S2, unofficial]. It addresses the performance bottleneck of regenerating mesh indices for every frame, even when the visible geometry has not changed. The technique leverages the discrete nature of voxel blocks to simplify visibility checks to coordinate comparisons.

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S2] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
