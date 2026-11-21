# Block Visibility Culling

Also called: Block Coordinate Change Detection, Block Rendering & Surface Orientation Culling, Camera-relative Block Culling, renderBlock

## Overview
Block Visibility Culling is a camera-relative optimization method that determines which voxel blocks require mesh updates or rendering by comparing the current camera position and orientation against the previous frame's state [S1, unofficial]. It avoids unnecessary computation by skipping index updates when the camera has not moved significantly or changed its primary viewing direction [S1, unofficial]. Unlike static face culling which only removes interior faces, this method operates at the block level, leveraging screen-space orientation to cull invisible faces (top, bottom, front, back) based on the player's position relative to the block [S2, unofficial].

## How it works
The method operates in two phases: a coarse-grained block-level skip check and a fine-grained face-level culling during rendering.

### Inputs and Outputs
- **Inputs**: Camera world position $(P_x, P_y, P_z)$, camera rotation angles (yaw $\theta$, pitch $\phi$), previous frame's camera block coordinates $(B_x^{prev}, B_y^{prev}, B_z^{prev})$ and greatest view direction `greatestDir`, block geometry data (position, size, visible face flags).
- **Outputs**: A boolean flag indicating whether the block's mesh indices need updating, and a set of visible faces for rendering.

### Data Structures
- **Camera State**: Stores current and previous integer block coordinates.
- **Block Face Flags**: An enumeration or bitmask array `BlockFace` with flags: `YNeg`, `XPos`, `YPos`, `XNeg`, `ZNeg`, `ZPos` [S2, unofficial]. The visibility of each face is stored in `block->visibleFaces[BlockFace::...]` [S2, unofficial].

### Steps

1. **Block Coordinate Check (Skip Optimization)**
   Compute the integer block coordinates of the camera by flooring its world position:
   $$ B_x = \lfloor P_x / \text{BLOCK\_SIZE} \rfloor, \quad B_y = \lfloor P_y / \text{BLOCK\_SIZE} \rfloor, \quad B_z = \lfloor P_z / \text{BLOCK\_SIZE} \rfloor $$
   Determine the greatest view direction `greatestDir` from the camera's rotation angles (rx, ry) [S1, unofficial].
   Compare $(B_x, B_y, B_z)$ and `greatestDir` with the previous frame's values. If both match, skip the index update process entirely [S1, unofficial].

2. **Surface Orientation Determination**
   For each block at chunk-relative coordinates $(x, y, z)$:
   - Calculate world coordinates:
     $$ \text{blockX} = \text{chunkX} + x \cdot \text{BLOCK\_SIZE} $$
     $$ \text{blockY} = \text{chunkY} + y \cdot \text{BLOCK\_SIZE} $$
     $$ \text{blockZBottom} = \text{chunkZ} + z \cdot \text{BLOCK\_SIZE} $$
     $$ \text{blockZTop} = \text{blockZBottom} + \text{BLOCK\_SIZE} $$
   - Determine `surfaceOrientation`:
     - If $P_z < \text{blockZBottom}$, `surfaceOrientation` = 1 (Floor/Ceiling view from below/above).
     - If $P_z > \text{blockZTop}$, `surfaceOrientation` = 2.
     - Otherwise, `surfaceOrientation` = 0 (Side view) [S2, unofficial].
   - Set rendering cycles: `cycles` = 2 if `surfaceOrientation` != 0, else 1 [S2, unofficial].

3. **Face Visibility Culling**
   During the rendering loop for each face $w$ (0 to 3 for sides):
   - Transform block corners to camera space:
     $$ x_1 = x_{\text{Bottom1}}[w] - P_x, \quad y_1 = y_{\text{Bottom1}}[w] - P_y $$
   - Swap $x_1/x_2$ and $y_1/y_2$ if `surfaceOrientation` == 1 [S2, unofficial].
   - Project to screen space using camera rotation:
     $$ \text{cs} = \cos(\theta \cdot \pi/180), \quad \text{sn} = \sin(\theta \cdot \pi/180) $$
     $$ \text{wallX}[w][0] = x_1 \cdot \text{cs} - y_1 \cdot \text{sn} $$
   - Check visibility flags before rendering:
     - If $w=0$ and `!block->visibleFaces[BlockFace::YNeg]`, skip.
     - If $w=1$ and `!block->visibleFaces[BlockFace::XPos]`, skip.
     - If $w=2$ and `!block->visibleFaces[BlockFace::YPos]`, skip.
     - If $w=3$ and `!block->visibleFaces[BlockFace::XNeg]`, skip [S2, unofficial].

### Parameters
- **Surface Scale Multiplier**: 120, used as `surfaceScale * 120` for tile calculation [S2, unofficial].
- **BLOCK_SIZE**: Standard voxel dimension (e.g., 16 units in some implementations) [S2, unofficial].

### Edge Cases and Pitfalls
- **Precision**: Floating-point comparisons for camera position should use epsilon tolerances if exact integer matches are not guaranteed due to sub-block movement [G].
- **Boundary Conditions**: When $P_z$ is exactly on a block boundary, `surfaceOrientation` logic must handle equality consistently to avoid flickering [G].
- **Degenerate Input**: If `xPos2 - xPos1` is 0, division by zero in horizontal step calculation must be guarded (e.g., set to 1) [S2, unofficial].

## Subtopics

### Voxel Face Visibility Check
A variant that determines visibility by computing the dot product between each face's center and its outward normal, marking it visible if the dot product is positive [S3, unofficial]. It defines normals for Front(0,0,1), Back(0,0,-1), Right(-1,0,0), Left(1,0,0), Top(0,1,0), Bottom(0,-1,0) and computes face centers as `cube_center + offset` [S3, unofficial]. The formula is $v_i = \mathbf{c}_i \cdot \mathbf{n}_i > 0$ where $\mathbf{c}_i$ is the face center and $\mathbf{n}_i$ is the outward normal [S3, unofficial]. A key pitfall is that this test uses the world origin (0,0,0) as the reference point, not the camera position, making it incorrect for arbitrary camera positions [S3, unofficial].

## Operating characteristics
- **Time Complexity**: $O(1)$ per block for the skip check; $O(F)$ for face culling where $F$ is the number of faces (typically 6). The skip check reduces average case complexity significantly in static scenes.
- **Memory Complexity**: $O(1)$ additional memory per block for storing previous camera coordinates and face flags.
- **Performance**: Reduces primitive count by avoiding mesh regeneration for unchanged blocks. In dynamic scenes, it minimizes CPU overhead by skipping index buffer updates [S1, unofficial].
- **Strengths**: Effective for static or slowly moving cameras; reduces draw calls and index buffer updates.
- **Limitations**: Ineffective for fast-moving cameras or dynamic scenes where the camera changes position every frame; does not handle occlusion by other geometry, only camera-relative visibility.

## Background
Originating from early voxel rendering engines and indie game development, this method is documented in repositories such as `kvxy/exolands` and `VidFurlan/Voxel1993` [S1, unofficial][S2, unofficial]. It is commonly used in real-time voxel engines where performance is critical and camera movement is often constrained or predictable.

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S2] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
[S3] anb2473/Python-Voxel-Rendering-Engine, https://github.com/anb2473/Python-Voxel-Rendering-Engine, fetched 2026-10-27 (unofficial)
