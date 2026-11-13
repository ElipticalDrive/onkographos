# Block Visibility Culling

Also called: Block Coordinate Change Detection, Block Rendering & Surface Orientation Culling, Camera-relative Block Culling, Visibility Culling via Normal Dot Product, renderBlock

## Overview
Block Visibility Culling is a coarse-grained optimization method that determines which voxel blocks require mesh updates or rendering based on the camera's position and orientation, avoiding unnecessary geometry generation when the viewing context has not changed significantly [S1, S3, unofficial]. It operates by comparing the camera's integer block coordinates and primary viewing direction against the previous frame's values to skip index updates, or by evaluating the dot product between surface normals and the view direction to cull invisible faces [S1, S2, unofficial]. Unlike sibling methods that perform fine-grained face-by-face culling based on neighbor occupancy, this method focuses on block-level visibility decisions driven by camera movement and orientation [S3, unofficial].

## How it works
**Inputs:**
- Camera world position $(P_x, P_y, P_z)$ and rotation angles $(\theta_x, \theta_y)$ [S1, S3, unofficial].
- Previous frame's camera block coordinates $(B_{x,prev}, B_{y,prev}, B_{z,prev})$ and greatest direction indicator [S1, unofficial].
- Block geometry data: chunk coordinates $(C_x, C_y, C_z)$, block local coordinates $(x, y, z)$, and block size $S$ [S3, unofficial].
- Surface vertex data for each block face [S2, S3, unofficial].

**Outputs:**
- Boolean flag indicating whether the block's mesh indices need updating [S1, unofficial].
- Set of visible faces for the block [S3, unofficial].

**Data Structures:**
- `BlockFace` enum or flags array storing visibility for the six faces: `YNeg`, `XPos`, `YPos`, `XNeg`, `ZNeg`, `ZPos` [S3, unofficial].
- Camera state variables: `player.x`, `player.y`, `player.z`, `player.angle` [S3, unofficial].

**Steps:**

1.  **Compute Camera Block Coordinates:**
    Calculate the integer block coordinates containing the camera by flooring the world position [S1, unofficial]:
    $$ B_x = \lfloor P_x / S \rfloor, \quad B_y = \lfloor P_y / S \rfloor, \quad B_z = \lfloor P_z / S \rfloor $$
    [S1, unofficial]

2.  **Check for Stasis (Optimization Skip):**
    Compare the current camera block coordinates $(B_x, B_y, B_z)$ and the greatest viewing direction with the previous frame's values. If both match, skip the index update process entirely [S1, unofficial].
    [S1, unofficial]

3.  **Determine Surface Orientation (Camera-Relative):**
    For each block face, determine its orientation relative to the camera [S3, unofficial]:
    - Calculate block boundaries:
      $$ B_{x,block} = C_x + x \cdot S $$
      $$ B_{y,block} = C_y + y \cdot S $$
      $$ B_{z,block, bottom} = C_z + z \cdot S $$
      $$ B_{z,block, top} = B_{z,block, bottom} + S $$
    - Determine `surfaceOrientation`:
      - $1$ if $P_z < B_{z,block, bottom}$ (camera below block)
      - $2$ if $P_z > B_{z,block, top}$ (camera above block)
      - $0$ otherwise (camera within vertical range)
    [S3, unofficial]

4.  **Transform to Camera Space:**
    Compute trigonometric components of the camera angle $\alpha$ (in degrees):
    $$ c_s = \cos(\alpha \cdot \pi / 180) $$
    $$ s_n = \sin(\alpha \cdot \pi / 180) $$
    [S3, unofficial]
    For each corner $w$ of the block's bottom face, compute the vector from the camera to the corner:
    $$ x_1 = X_{bottom,1}[w] - P_x $$
    $$ y_1 = Y_{bottom,1}[w] - P_y $$
    [S3, unofficial]
    Swap $x_1/x_2$ and $y_1/y_2$ if `surfaceOrientation` is $1$ [S3, unofficial].
    Project to camera-relative coordinates:
    $$ wallX[w][0] = x_1 \cdot c_s - y_1 \cdot s_n $$
    [S3, unofficial]

5.  **Compute View Direction and Average Normal:**
    Calculate the view vector components $(v_x, v_y, v_z)$ from the camera's rotation angles $(\theta_x, \theta_y)$ [S1, unofficial].
    For each surface, calculate the average normal $\mathbf{n}_{avg}$ as the mean of its vertex normals, then normalize it [S2, unofficial]:
    $$ \mathbf{n}_{avg} = \frac{\sum \mathbf{n}_i}{|\sum \mathbf{n}_i|} $$
    [S2, unofficial]
    Compute the view direction $\mathbf{v}$ from the surface center to the camera position and normalize it [S2, unofficial].

6.  **Visibility Check via Dot Product:**
    Compute the dot product $D = \mathbf{n}_{avg} \cdot \mathbf{v}$ [S2, unofficial].
    If $D > 0$, the surface is considered visible [S2, unofficial].
    Alternatively, use pre-computed `visibleFaces` flags based on orientation checks to skip rendering invisible faces [S3, unofficial]:
    - If `surfaceOrientation` is $1$ (below), check `block->visibleFaces[BlockFace::YNeg]`.
    - If `surfaceOrientation` is $2$ (above), check `block->visibleFaces[BlockFace::YPos]`.
    - For side faces, check corresponding X/Z flags based on camera position relative to block center [S3, unofficial].

7.  **Render or Update Indices:**
    If the face is visible according to the checks above, add the face indices to the mesh buffer [S1, S3, unofficial]. Otherwise, skip the face [S3, unofficial].

**Parameters:**
- `surfaceScale`: Multiplier for floor/ceiling projection, typically $120$ [S3, unofficial].
- `BLOCK_SIZE`: Standard voxel dimension, typically $16$ or $1$ depending on implementation [S3, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point comparisons for camera position and block boundaries must handle small epsilon errors to avoid flickering when the camera is exactly on a block boundary [G].
- **Degenerate Input:** If `xPos2 - xPos1` is zero, handle division by zero in texture coordinate calculations by setting the step to a small non-zero value (e.g., $1$) [S3, unofficial].
- **Boundary Conditions:** Ensure that out-of-bounds neighbors are treated as empty space to correctly identify exposed faces [S2, unofficial].

## Subtopics

### [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md)
Determines visibility of individual faces within a block by evaluating the geometric relationship between face normals and a reference direction. It operates at the fine-grained face level, whereas Block Visibility Culling operates at the coarse block level based on camera movement. Choose this when fine-grained control over individual face rendering is required.

## Operating characteristics
- **Time Complexity:** $O(1)$ per block for the stasis check (coordinate comparison). $O(F)$ for face visibility checks where $F$ is the number of faces (typically 6).
- **Memory Complexity:** $O(1)$ additional memory per block for storing visibility flags and previous camera state.
- **Measured Performance:** Reduces unnecessary mesh updates when the camera is stationary or moving slowly within the same block [S1, unofficial].
- **Scaling:** Scales linearly with the number of blocks in the view frustum, but the stasis check provides significant savings in static scenes.
- **Strengths:** Simple to implement, low computational overhead, effective for static or slow-moving cameras.
- **Limitations:** Does not account for occlusion by other blocks (requires additional depth testing or ray casting). Less effective in highly dynamic scenes with rapid camera movement.
- **When to Use:** Ideal for voxel engines where camera movement is relatively slow or static, and mesh generation is expensive.
- **When Not to Use:** Not suitable for scenes with rapid camera movement or complex occlusion requirements where fine-grained face culling is necessary.

## Background
Block Visibility Culling originated in early voxel rendering implementations, such as those found in indie voxel engines and educational projects [S1, S3, unofficial]. It is based on the observation that rendering geometry for blocks that are not visible to the camera is a waste of resources. The method has been adopted in various open-source voxel renderers to optimize performance by reducing the number of draw calls and mesh updates [S1, S3, unofficial].

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S2] matthxie/NeRF-Renderer, https://github.com/matthxie/NeRF-Renderer, fetched 2026-10-27 (unofficial)
[S3] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
