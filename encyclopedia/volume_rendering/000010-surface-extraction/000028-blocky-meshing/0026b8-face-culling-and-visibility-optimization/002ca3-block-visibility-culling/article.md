# Block Visibility Culling

Also called: Block Coordinate Change Detection, Block Rendering & Surface Orientation Culling, Camera-relative Block Culling, renderBlock

## Overview
Block Visibility Culling determines which voxel blocks require mesh updates or rendering based on the camera's position and rotation, avoiding unnecessary geometry generation when the viewing direction or block-relative position has not changed significantly [S1, S2, unofficial]. Unlike its parent [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md), which relies on local voxel topology (neighbor occupancy) to cull faces, this method operates at the coarse block level using camera-relative geometric checks to skip entire blocks or specific surface orientations [S1, S2, unofficial]. It is distinct from [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md) by operating on the block aggregate rather than individual faces within a block [S1, S2, unofficial].

## How it works
**Inputs:** Camera world position $(P_x, P_y, P_z)$, camera rotation angle $\theta$ (in degrees), block grid coordinates $(chunkX, chunkY, chunkZ)$, block dimensions $BLOCK\_SIZE$, and previous frame's camera block coordinates and greatest viewing direction.
**Outputs:** A set of blocks and faces marked for rendering; skipped blocks if no change is detected.

**Data Structures:**
*   `BlockFace`: An enumeration or bitmask with flags `YNeg`, `XPos`, `YPos`, `XNeg`, `ZNeg`, `ZPos` representing the six faces of a block [S2, unofficial].
*   `visibleFaces`: An array or bitmask associated with each block, storing which faces are potentially visible based on neighbor occupancy [S2, unofficial].
*   `blocksData`: A map or array storing indices for each block's mesh data [S1, unofficial].

**Steps:**

1.  **Compute Camera Block Coordinates:**
    Calculate the integer block coordinates containing the camera by flooring the world position:
    $$
    camBlockX = \lfloor P_x / BLOCK\_SIZE \rfloor
    $$
    $$
    camBlockY = \lfloor P_y / BLOCK\_SIZE \rfloor
    $$
    $$
    camBlockZ = \lfloor P_z / BLOCK\_SIZE \rfloor
    $$
    [S1, unofficial]

2.  **Check for Stasis (Optimization):**
    Compare the current $(camBlockX, camBlockY, camBlockZ)$ and the greatest viewing direction (derived from camera rotation) with the values from the previous frame. If both the block coordinates and the greatest direction match, skip the index update process entirely for this frame [S1, unofficial].

3.  **Determine Surface Orientation:**
    For each block at $(chunkX, chunkY, chunkZ)$, calculate the block's vertical bounds:
    $$
    blockZ_{bottom} = chunkZ \times BLOCK\_SIZE
    $$
    $$
    blockZ_{top} = blockZ_{bottom} + BLOCK\_SIZE
    $$
    Determine the surface orientation relative to the camera's vertical position $P_z$:
    *   If $P_z < blockZ_{bottom}$, orientation = 1 (Camera is below the block).
    *   If $P_z > blockZ_{top}$, orientation = 2 (Camera is above the block).
    *   Otherwise, orientation = 0 (Camera is within the block's vertical range) [S2, unofficial].

4.  **Calculate Cycles:**
    Set the number of rendering cycles:
    $$
    cycles = 2 \quad \text{if orientation} \neq 0
    $$
    $$
    cycles = 1 \quad \text{if orientation} = 0
    $$
    This determines whether top/bottom faces are processed separately [S2, unofficial].

5.  **Transform to Camera Space:**
    Compute trigonometric components for the camera angle $\theta$:
    $$
    cs = \cos(\theta \times \pi / 180)
    $$
    $$
    sn = \sin(\theta \times \pi / 180)
    $$
    For each corner $w$ of the block's base (0 to 3), calculate the bottom corner coordinates in world space:
    $$
    x_{bottom}[w] = chunkX \times BLOCK\_SIZE + x \times BLOCK\_SIZE
    $$
    $$
    y_{bottom}[w] = chunkY \times BLOCK\_SIZE + y \times BLOCK\_SIZE
    $$
    Transform to camera-relative space:
    $$
    x_1 = x_{bottom}[w] - P_x
    $$
    $$
    y_1 = y_{bottom}[w] - P_y
    $$
    If orientation == 1 (camera below), swap $x_1/x_2$ and $y_1/y_2$ to adjust for viewing angle [S2, unofficial].
    Project to screen-space-like coordinates:
    $$
    wallX[w][0] = x_1 \times cs - y_1 \times sn
    $$
    [S2, unofficial]

6.  **Face Visibility Check:**
    Before rendering a specific face $w$ (0 to 3 for side walls), check the block's `visibleFaces` flag corresponding to that face [S2, unofficial]:
    *   $w=0$: Check `BlockFace::YNeg`
    *   $w=1$: Check `BlockFace::XPos`
    *   $w=2$: Check `BlockFace::YPos`
    *   $w=3$: Check `BlockFace::XNeg`
    If the flag is false, skip rendering this face [S2, unofficial].

7.  **Render/Update Indices:**
    If the face is visible and passes the stasis check, generate or update the mesh indices for the face. If the block has no visible faces left, remove its entry from `blocksData` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def block_visibility_culling(camera_pos, camera_angle_deg, block_chunk, block_size, prev_cam_block, prev_greatest_dir, block_visible_faces):
    """
    Determines if a block needs rendering based on camera position and orientation.
    
    Args:
        camera_pos: tuple (px, py, pz)
        camera_angle_deg: float, camera rotation in degrees
        block_chunk: tuple (cx, cy, cz), block's chunk/grid coordinates
        block_size: int, size of one block
        prev_cam_block: tuple, camera block coords from previous frame
        prev_greatest_dir: int, greatest viewing direction from previous frame
        block_visible_faces: dict or bitmask, keys are BlockFace enums
        
    Returns:
        bool: True if block needs update/render, False if skipped
    """
    px, py, pz = camera_pos
    cx, cy, cz = block_chunk
    
    # Step 1: Compute Camera Block Coordinates
    cam_block_x = math.floor(px / block_size)
    cam_block_y = math.floor(py / block_size)
    cam_block_z = math.floor(pz / block_size)
    
    # Step 2: Check for Stasis
    # Note: 'greatest_dir' logic is simplified here as exact derivation from angle is not fully specified in sources
    # Assuming a helper function exists to determine greatest_dir from camera_angle_deg
    current_greatest_dir = get_greatest_direction(camera_angle_deg) 
    
    if (cam_block_x, cam_block_y, cam_block_z) == prev_cam_block and current_greatest_dir == prev_greatest_dir:
        return False  # Skip update
    
    # Step 3: Determine Surface Orientation
    block_z_bottom = cz * block_size
    block_z_top = block_z_bottom + block_size
    
    if pz < block_z_bottom:
        orientation = 1
    elif pz > block_z_top:
        orientation = 2
    else:
        orientation = 0
        
    # Step 4: Calculate Cycles
    cycles = 2 if orientation != 0 else 1
    
    # Step 5 & 6: Transform and Check Faces
    # This part is typically handled in the rendering loop per face
    # We return True to indicate the block should be processed
    return True

def get_greatest_direction(angle_deg):
    """Helper to determine primary viewing direction from angle."""
    # Simplified logic: 0=Front, 1=Right, 2=Back, 3=Left based on quadrant
    # Exact mapping depends on coordinate system, not fully specified in sources [S1, unofficial]
    return 0 # Placeholder for implementation detail [G]
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point comparisons for camera position should use epsilon if exact integer block boundaries are not guaranteed [G].
*   **Degenerate Input:** If `xPos2 - xPos1` is 0 during projection, handle division by zero by setting distance to 1 [S2, unofficial].
*   **Orientation Swapping:** The swap of $x_1/x_2$ and $y_1/y_2$ when orientation is 1 is critical for correct screen-space projection; omitting it causes visual artifacts [S2, unofficial].

## Subtopics

### [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md)
Determines visibility of individual faces within a block based on geometric relationship between face normals and reference direction. Operates at fine-grained face level, unlike the coarse block-level check of this method. Choose when per-face culling is needed within a block that is already determined to be visible.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per block for the stasis check. $O(N)$ for $N$ visible blocks. The trigonometric calculations are constant time per block.
*   **Memory Complexity:** $O(1)$ additional memory per block for storing `visibleFaces` and previous frame state.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Significantly reduces mesh update overhead when the camera is stationary or moving slowly within the same block grid cell. Simple to implement.
*   **Limitations:** Does not cull faces based on occlusion or distant visibility. Relies on the assumption that camera movement is the primary driver for mesh updates.
*   **When to Use:** Real-time voxel engines where camera movement is frequent but often incremental, and mesh generation is expensive.
*   **When Not to Use:** Static scenes where camera movement is irrelevant, or scenes with high-frequency camera jitter where the stasis check rarely triggers.

## Background
Originating from early voxel engine implementations, this method is documented in unofficial repositories such as `kvxy/exolands` [S1, unofficial] and `VidFurlan/Voxel1993` [S2, unofficial]. It reflects a practical optimization strategy for 3D voxel rendering where reducing the number of triangles submitted to the graphics pipeline is critical for performance on limited hardware.

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S2] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
