# Block Visibility Culling

Also called: Block Coordinate Change Detection, Block Rendering & Surface Orientation Culling, Camera-relative Block Culling, renderBlock

## Overview
Block Visibility Culling determines which voxel blocks require mesh updates or rendering based on the camera's position and rotation, avoiding unnecessary geometry generation when the viewing direction has not changed significantly [S1, S2, unofficial]. Unlike its sibling [Face Culling](8735-face-culling.md), which relies on local voxel adjacency to hide internal faces, this method operates at the coarse block level by comparing the current camera state against the previous frame's state to skip redundant processing [S1, unofficial]. It further refines visibility by calculating screen-space orientation relative to the player to cull faces that are geometrically facing away from the viewer [S2, unofficial].

## How it works
**Inputs:**
- Camera world position $(P_x, P_y, P_z)$.
- Camera rotation angle $\theta$ (in degrees, typically yaw).
- Previous frame's camera block coordinates $(C_{x,prev}, C_{y,prev}, C_{z,prev})$ and greatest viewing direction $D_{prev}$.
- Block grid coordinates $(x, y, z)$ and chunk offset $(chunkX, chunkY, chunkZ)$.
- Block size $BLOCK\_SIZE$ (typically 16 units).
- Block face visibility flags $V \in \{YNeg, XPos, YPos, XNeg, ZNeg, ZPos\}$ [S2, unofficial].

**Outputs:**
- Boolean flag indicating if the block's mesh needs regeneration.
- Set of visible faces for the block.

**Data Structures:**
- `BlockFace` enum or bitmask array storing visibility flags for the six faces of a block [S2, unofficial].
- Camera state tuple: $(blockX, blockY, blockZ, greatestDir)$ [S1, unofficial].

**Steps:**

1. **Compute Camera Block Coordinates:**
   Calculate the integer block coordinates containing the camera by flooring the world position:
   $$
   C_x = \lfloor P_x \rfloor, \quad C_y = \lfloor P_y \rfloor, \quad C_z = \lfloor P_z \rfloor
   $$
   [S1, unofficial]

2. **Check for State Change (Optimization Guard):**
   Determine the current greatest viewing direction $D_{curr}$ based on camera rotation angles $(rx, ry)$ to form the view vector $(v_x, v_y, v_z)$ [S1, unofficial]. Compare $(C_x, C_y, C_z, D_{curr})$ with the previous frame's values $(C_{x,prev}, C_{y,prev}, C_{z,prev}, D_{prev})$. If all components match, skip the index update process entirely [S1, unofficial].

3. **Calculate Block Geometry and Orientation:**
   For a block at grid coordinates $(x, y, z)$ within a chunk:
   $$
   \begin{aligned}
   B_x &= chunkX + x \cdot BLOCK\_SIZE \\
   B_y &= chunkY + y \cdot BLOCK\_SIZE \\
   B_{z,bot} &= chunkZ + z \cdot BLOCK\_SIZE \\
   B_{z,top} &= B_{z,bot} + BLOCK\_SIZE
   \end{aligned}
   $$
   [S1, unofficial]
   
   Determine surface orientation relative to the camera's vertical position:
   $$
   \text{orientation} = 
   \begin{cases} 
   1 & \text{if } P_z < B_{z,bot} \quad (\text{Looking up}) \\
   2 & \text{if } P_z > B_{z,top} \quad (\text{Looking down}) \\
   0 & \text{otherwise} \quad (\text{Level view})
   \end{cases}
   $$
   [S1, unofficial]

4. **Transform to Camera Space:**
   Compute trigonometric components:
   $$
   cs = \cos(\theta \cdot \pi / 180), \quad sn = \sin(\theta \cdot \pi / 180)
   $$
   [S1, unofficial]
   
   Calculate bottom corner coordinates relative to the camera:
   $$
   \begin{aligned}
   x_1 &= x_{bottom1}[w] - P_x \\
   y_1 &= y_{bottom1}[w] - P_y
   \end{aligned}
   $$
   [S1, unofficial]
   
   Swap $x_1/x_2$ and $y_1/y_2$ if $\text{orientation} == 1$ [S1, unofficial].

5. **Project to Screen Space:**
   Compute the projected wall X coordinate:
   $$
   wallX[w][0] = x_1 \cdot cs - y_1 \cdot sn
   $$
   [S1, unofficial]
   
   Calculate vertical steps and positions. If $\text{orientation} \neq 0$, set rendering cycles to 2; otherwise, 1 [S1, unofficial].

6. **Apply Face Visibility Flags:**
   Before rendering each face $w \in \{0, 1, 2, 3\}$, check the block's visibility flags [S2, unofficial]:
   - If $w=0$ (Bottom), skip if $\neg V[YNeg]$.
   - If $w=1$ (Right), skip if $\neg V[XPos]$.
   - If $w=2$ (Top), skip if $\neg V[YPos]$.
   - If $w=3$ (Left), skip if $\neg V[XNeg]$.
   [S2, unofficial]

**Parameters:**
- `surfaceScale`: Multiplier for floor/ceiling projection, typically 120 [S2, unofficial].
- `BLOCK_SIZE`: Standard voxel dimension, typically 16 [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** When calculating vertical distances for projection, if $z=0$, use a small epsilon (e.g., 0.0001) to avoid division by zero [S2, unofficial].
- **Boundary Conditions:** Ensure $rx$ and $ry$ indices are within valid texture map bounds (e.g., $0 < rx < 5$) before accessing pixel data [S2, unofficial].
- **Degenerate Input:** If the camera is exactly on a block boundary, floor operations determine the block index; consistency with the neighbor's culling logic is required to prevent Z-fighting or gaps [G].

## Subtopics

### [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md)
Determines visibility of individual voxel faces within a block by evaluating geometric relationships between face normals and the reference direction. It operates at the fine-grained face level, whereas Block Visibility Culling operates at the coarse block level based on camera movement. Choose this when fine-grained optimization within a single block is needed.

## Operating characteristics
- **Time Complexity:** $O(1)$ per block for the state check; $O(N)$ for mesh generation only when the state changes. The state check avoids $O(N)$ mesh regeneration entirely if the camera has not moved significantly [S1, unofficial].
- **Memory Complexity:** $O(1)$ additional memory per block to store visibility flags and previous camera state [S2, unofficial].
- **Measured Performance:** In implementations like [S2, unofficial], this method reduces draw calls by skipping invisible faces (e.g., floors when looking up) and avoids redundant mesh updates. Performance gains are most significant in large, static scenes with frequent camera rotation but little positional change.
- **Strengths:** Eliminates redundant geometry generation; simple to implement; effective for first-person perspectives.
- **Limitations:** Does not cull faces based on occlusion by other blocks (requires [Face Culling](8735-face-culling.md) or [Mask Projection Culling](6836-mask-projection-culling.md)); less effective if the camera moves continuously.
- **When to Use:** First-person voxel games or viewers where camera movement is frequent but scene geometry is static.
- **When Not to Use:** Third-person views with large camera swings, or dynamic scenes where blocks are frequently added/removed (requires incremental updates).

## Background
Block Visibility Culling originated in early voxel rendering engines, particularly those simulating first-person perspectives in constrained environments. It is prominently featured in unofficial implementations such as `exolands` [S1, unofficial] and `Voxel1993` [S2, unofficial], which emulate retro-style rendering techniques. The method leverages the observation that in many voxel applications, the mesh does not change unless the camera's relative position to the block grid changes significantly.

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S2] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
