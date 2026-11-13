# Block Visibility Culling

Also called: Block Coordinate Change Detection, Block Rendering & Surface Orientation Culling, Camera-relative Block Culling, Visibility Culling via Normal Dot Product, renderBlock

## Overview
Block Visibility Culling determines which voxel blocks require mesh regeneration or rendering updates based on the camera's position and orientation, avoiding unnecessary geometry processing when the viewer has not moved significantly or changed their primary viewing direction [S1, unofficial][S3, unofficial]. It operates at the coarse block level by comparing current camera coordinates and viewing direction against previous frame values, serving as a high-level optimization that complements finer-grained face culling [S3, unofficial]. Unlike [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md), which evaluates individual surface normals, this method evaluates the visibility of entire blocks based on camera-relative orientation [S3, unofficial].

## How it works
**Inputs:**
- Camera world position $(P_x, P_y, P_z)$.
- Camera rotation angles (e.g., yaw $\theta$ in degrees or radians).
- Previous frame's camera block coordinates $(C_{x,prev}, C_{y,prev}, C_{z,prev})$ and viewing direction indicator (e.g., `greatestDir`).
- Block size $B$ (typically 16 or 32 units, depending on implementation) [S3, unofficial].
- Chunk coordinates $(ChunkX, ChunkY, ChunkZ)$ defining the current spatial partition [S3, unofficial].

**Outputs:**
- A boolean flag indicating whether the block's mesh indices need updating.
- A visibility mask for the block's faces, stored in an array or enum `BlockFace` with flags: `YNeg`, `XPos`, `YPos`, `XNeg`, `ZNeg`, `ZPos` [S3, unofficial].

**Data Structures:**
- `BlockFace` enum: Represents the six faces of a voxel block.
- `visibleFaces`: An array or bitmask storing the visibility status of each face for a given block [S3, unofficial].

**Steps:**
1. **Compute Camera Block Coordinates:**
   Calculate the integer block coordinates containing the camera by flooring the world position divided by the block size:
   $$
   C_x = \lfloor P_x / B \rfloor, \quad C_y = \lfloor P_y / B \rfloor, \quad C_z = \lfloor P_z / B \rfloor
   $$
   [S1, unofficial]

2. **Check for Significant Change:**
   Compare the current block coordinates $(C_x, C_y, C_z)$ and the primary viewing direction (`greatestDir`) with the values from the previous frame. If both the block coordinates and the viewing direction match the previous frame, skip the index update process for this block [S1, unofficial].

3. **Calculate Surface Orientation (for face culling):**
   For each block, determine the relative position of the camera to the block's vertical extent to classify surface orientation:
   - Calculate block vertical bounds:
     $$
     Z_{bottom} = ChunkZ \times B + z \times B
     $$
     $$
     Z_{top} = Z_{bottom} + B
     $$
   - Determine `surfaceOrientation`:
     - If $P_z < Z_{bottom}$, `surfaceOrientation` = 1 (Camera is below the block).
     - If $P_z > Z_{top}$, `surfaceOrientation` = 2 (Camera is above the block).
     - Otherwise, `surfaceOrientation` = 0 (Camera is within the vertical range) [S3, unofficial].

4. **Compute View Direction and Dot Product:**
   - Calculate the view vector components $(v_x, v_y, v_z)$ from the camera's rotation angles (e.g., $v_x = \cos(\theta), v_y = \sin(\theta)$) [S1, unofficial].
   - For each surface of the block, calculate the center as the mean of its vertices.
   - Calculate the average normal $\mathbf{n}$ as the mean of its vertex normals and normalize it [S2, unofficial].
   - Compute the view direction $\mathbf{v}$ from the surface center to the camera position and normalize it [S2, unofficial].
   - Calculate the dot product $d = \mathbf{n} \cdot \mathbf{v}$.
   - If $d > 0$, the surface is considered visible; otherwise, it is culled [S2, unofficial].

5. **Transform to Camera Space (for rendering):**
   - Compute trigonometric values: $c_s = \cos(\theta \cdot \pi/180)$, $s_n = \sin(\theta \cdot \pi/180)$ [S3, unofficial].
   - Transform block corners to camera space:
     $$
     x_1 = x_{bottom1}[w] - P_x, \quad y_1 = y_{bottom1}[w] - P_y
     $$
   - Swap $x_1/x_2$ and $y_1/y_2$ if `surfaceOrientation` == 1 [S3, unofficial].
   - Project to screen coordinates:
     $$
     wallX[w][0] = x_1 \cdot c_s - y_1 \cdot s_n
     $$
     [S3, unofficial]

6. **Apply Visibility Flags:**
   Before rendering, check the `visibleFaces` array. For example, if `w == 0` (bottom face), skip rendering if `!block->visibleFaces[BlockFace::YNeg]` [S3, unofficial].

**Parameters:**
- `surfaceScale`: A multiplier for floor/ceiling projection, typically 120 (used as `surfaceScale * 120` for tile calculation) [S3, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point errors in dot product calculations may cause flickering on boundary cases where the normal is perpendicular to the view direction. A small epsilon threshold may be needed.
- **Degenerate Input:** If the camera is exactly on the boundary of a block, the block coordinate calculation must handle the floor function consistently to avoid double-rendering or gaps.
- **Static Camera:** If the camera does not move, the method correctly skips updates, but dynamic scenes require careful management of the "previous frame" state.

## Subtopics

### [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md)
Determines visibility of individual voxel faces within a block by evaluating geometric relationships between face normals and a reference direction. It operates at a finer granularity than Block Visibility Culling, which works at the block level based on camera movement. Choose this when fine-grained control over individual face rendering is required within a block.

## Operating characteristics
- **Time Complexity:** $O(1)$ per block for the coordinate check and orientation calculation. The dot product calculation is $O(F)$ where $F$ is the number of faces per block (typically 6).
- **Memory Complexity:** $O(1)$ per block for storing visibility flags.
- **Measured Performance:** Significant reduction in mesh update overhead when the camera is static or moving slowly within the same block [S1, unofficial].
- **Strengths:** Simple to implement; effective for static or slow-moving cameras; reduces CPU load for mesh generation.
- **Limitations:** Does not account for occlusion by other blocks; less effective for fast-moving cameras or complex scenes with many small blocks.
- **When to Use:** Ideal for voxel-based games or applications where the camera moves in discrete steps or remains relatively static for periods.
- **When Not to Use:** Not suitable for dynamic scenes with rapid camera movement or where fine-grained occlusion culling is required.

## Background
Block Visibility Culling originated in early voxel-based rendering systems, such as those found in indie voxel games and educational graphics projects. It is commonly implemented in JavaScript-based voxel engines (e.g., Exolands) and C++ voxel renderers (e.g., Voxel1993) to optimize rendering performance by reducing the number of triangles submitted to the graphics pipeline [S1, unofficial][S3, unofficial]. The method leverages the regular structure of voxel grids to simplify visibility checks to coordinate comparisons and basic trigonometry.

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S2] matthxie/NeRF-Renderer, https://github.com/matthxie/NeRF-Renderer, fetched 2026-10-27 (unofficial)
[S3] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
