# Block Visibility Culling

Also called: Block Coordinate Change Detection, Block Rendering & Surface Orientation Culling, Camera-relative Block Culling, renderBlock

## Overview
Block Visibility Culling determines which voxel blocks require mesh updates or rendering based on the camera's position and rotation, avoiding unnecessary geometry generation when the camera has not moved significantly or changed its primary viewing direction [S1, S2, unofficial]. It operates at the coarse block level by comparing current camera coordinates against previous frame values to skip index updates entirely, or by calculating screen-space orientation to cull invisible faces within a block [S1, S2, unofficial]. Unlike its sibling [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md), which evaluates individual face normals for visibility, this method makes decisions based on block-level camera-relative state [S1, S2, unofficial].

## How it works
**Inputs:**
- Camera world position $(P_x, P_y, P_z)$ and rotation angle $\theta$ (in degrees) [S1, S2, unofficial].
- Previous frame's camera block coordinates $(C_{x}^{prev}, C_{y}^{prev}, C_{z}^{prev})$ and greatest viewing direction $D^{prev}$ [S1, unofficial].
- Block grid parameters: chunk origin $(ChunkX, ChunkY, ChunkZ)$, block indices $(x, y, z)$, and block size $BLOCK\_SIZE$ [S2, unofficial].
- Block visibility flags: an array or enum `BlockFace` with flags `YNeg`, `XPos`, `YPos`, `XNeg`, `ZNeg`, `ZPos` [S2, unofficial].

**Outputs:**
- Boolean flag indicating whether the block's mesh indices need updating [S1, unofficial].
- Set of visible faces for the block [S2, unofficial].

**Data Structures:**
- `BlockFace`: Enum or bitmask representing the six faces of a block.
- `visibleFaces`: Array or bitmask storing which faces are currently visible [S2, unofficial].
- `blocksData`: Map or array storing triangle indices for each block [S1, unofficial].

**Steps:**

1. **Compute Camera Block Coordinates:**
   Calculate the integer block coordinates containing the camera by flooring the world position:
   $$
   C_x = \lfloor P_x \rfloor, \quad C_y = \lfloor P_y \rfloor, \quad C_z = \lfloor P_z \rfloor
   $$
   [S1, unofficial]

2. **Check for Stasis (Optimization):**
   Determine the current greatest viewing direction $D$ based on camera rotation (e.g., which axis has the largest absolute component of the view vector) [S1, unofficial].
   If $C_x = C_{x}^{prev}$ AND $C_y = C_{y}^{prev}$ AND $C_z = C_{z}^{prev}$ AND $D = D^{prev}$, skip the index update process entirely and return [S1, unofficial].
   Otherwise, update $C_{x}^{prev}, C_{y}^{prev}, C_{z}^{prev}, D^{prev}$ to current values [S1, unofficial].

3. **Calculate Block World Bounds:**
   For a block at local indices $(x, y, z)$ within a chunk:
   $$
   B_x = ChunkX + x \cdot BLOCK\_SIZE
   $$
   $$
   B_y = ChunkY + y \cdot BLOCK\_SIZE
   $$
   $$
   B_{z\_bottom} = ChunkZ + z \cdot BLOCK\_SIZE
   $$
   $$
   B_{z\_top} = B_{z\_bottom} + BLOCK\_SIZE
   $$
   [S2, unofficial]

4. **Determine Surface Orientation:**
   Evaluate the camera's vertical position relative to the block to determine the primary viewing orientation:
   $$
   \text{surfaceOrientation} = 
   \begin{cases} 
   1 & \text{if } P_z < B_{z\_bottom} \quad (\text{Looking Up}) \\
   2 & \text{if } P_z > B_{z\_top} \quad (\text{Looking Down}) \\
   0 & \text{otherwise} \quad (\text{Looking Straight})
   \end{cases}
   $$
   [S2, unofficial]

5. **Compute Trigonometric Constants:**
   Convert the camera angle $\theta$ from degrees to radians and compute sine and cosine:
   $$
   c_s = \cos\left(\theta \cdot \frac{\pi}{180}\right)
   $$
   $$
   s_n = \sin\left(\theta \cdot \frac{\pi}{180}\right)
   $$
   [S2, unofficial]

6. **Transform Block Corners to Camera Space:**
   For each of the four vertical edges (indexed $w = 0$ to $3$), compute the bottom corner coordinates in world space $(x_{bottom1}[w], y_{bottom1}[w])$ [S2, unofficial].
   Transform to camera-relative space:
   $$
   x_1 = x_{bottom1}[w] - P_x
   $$
   $$
   y_1 = y_{bottom1}[w] - P_y
   $$
   [S2, unofficial]

7. **Apply Orientation Swap:**
   If $\text{surfaceOrientation} == 1$ (Looking Up), swap $x_1$ and $y_1$ (and corresponding top coordinates if applicable) to correct for the inverted view [S2, unofficial].

8. **Project to Screen Space (Partial):**
   Compute the projected X coordinate for the wall face:
   $$
   wallX[w][0] = x_1 \cdot c_s - y_1 \cdot s_n
   $$
   [S2, unofficial]
   *Note: Full projection to screen Y involves distance calculations and perspective division, often handled in subsequent rendering passes [S2, unofficial].*

9. **Cull Invisible Faces:**
   Before rendering or adding indices for a specific face $w$ (where $w=0$ is YNeg, $w=1$ is XPos, $w=2$ is YPos, $w=3$ is XNeg), check the `visibleFaces` flag [S2, unofficial]:
   - If $w=0$ and `!block->visibleFaces[BlockFace::YNeg]`, skip.
   - If $w=1$ and `!block->visibleFaces[BlockFace::XPos]`, skip.
   - If $w=2$ and `!block->visibleFaces[BlockFace::YPos]`, skip.
   - If $w=3$ and `!block->visibleFaces[BlockFace::XNeg]`, skip.
   [S2, unofficial]

10. **Update Mesh Indices:**
    If the face is visible, add the corresponding triangle indices to the mesh buffer. If the block has no visible faces, remove its entry from `blocksData` and update block positions [S1, unofficial].

**Parameters:**
- `surfaceScale`: Multiplier for floor/ceiling projection, typically 120 [S2, unofficial].
- `BLOCK_SIZE`: Size of a voxel block in world units (implementation dependent, often 16 or 32) [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point comparisons for camera position stasis should use exact integer block coordinates to avoid jitter [S1, unofficial].
- **Boundary Conditions:** When $P_z$ is exactly on $B_{z\_bottom}$ or $B_{z\_top}$, the orientation is 0 (Looking Straight) [S2, unofficial].
- **Degenerate Input:** If $xPos2 - xPos1 = 0$ in projection calculations, use a default step of 1 to avoid division by zero [S2, unofficial].

## Subtopics

### [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md)
Determines visibility of individual faces within a block based on geometric normals and view direction. Choose this when fine-grained face culling is needed within a block that has already been deemed visible.

## Operating characteristics
- **Time Complexity:** $O(1)$ per block for the stasis check. $O(N)$ for blocks requiring update, where $N$ is the number of visible faces.
- **Memory Complexity:** $O(1)$ additional memory per block for storing visibility flags and previous camera state.
- **Performance:** Significantly reduces draw calls and mesh generation overhead when the camera is stationary or moving slowly within the same block [S1, unofficial].
- **Strengths:** Simple to implement; effective for static scenes or slow camera movement [S1, unofficial].
- **Limitations:** Does not cull faces based on occlusion or backfacing geometry within the block; relies on coarse block-level decisions [S1, S2, unofficial].
- **When to use:** Ideal for voxel engines where camera movement is discrete or slow, and block-level updates are expensive.
- **When not to use:** In fast-paced scenes with rapid camera rotation where block-level culling is insufficient, or when fine-grained face culling is required.

## Background
Block Visibility Culling originated in early voxel-based games and engines, particularly those using software rendering or limited hardware resources. It is implemented in projects like `exolands` [S1, unofficial] and `Voxel1993` [S2, unofficial], which emulate retro-style voxel rendering techniques. The method leverages the observation that small camera movements within a single block do not change the set of visible faces, allowing for significant optimization by skipping mesh regeneration.

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S2] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
