# Block Visibility Culling

Also called: Block Coordinate Change Detection, Block Rendering & Surface Orientation Culling, Camera-relative Block Culling, renderBlock

## Overview
Block Visibility Culling determines which voxel blocks require mesh updates or rendering by comparing the current camera position and orientation against the previous frame's state [S1, unofficial][S2, unofficial]. It solves the problem of redundant geometry generation when the camera has not moved significantly or changed its primary viewing direction relative to the block grid [S1, unofficial]. Unlike sibling methods that operate on individual voxel faces or post-generated meshes, this method operates at the coarse block level, skipping entire block processing if the camera remains within the same integer block coordinates and maintains the same dominant viewing axis [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera world position $(P_x, P_y, P_z)$, camera rotation angle $\theta$ (in degrees), previous frame's camera block coordinates $(B_{prev\_x}, B_{prev\_y}, B_{prev\_z})$, previous frame's greatest viewing direction $D_{prev}$, block size $S$, and block visibility flags.
*   **Outputs:** Boolean flag indicating whether to skip index update/rendering for the current frame; updated block coordinates and viewing direction for the next frame.

**Data Structures:**
*   **Block Face Visibility:** An array or enum `BlockFace` with flags: `YNeg`, `XPos`, `YPos`, `XNeg`, `ZNeg`, `ZPos` [S2, unofficial]. These flags determine which of the six faces of a block are visible to the viewer.
*   **Camera State:** Stores integer block coordinates and the dominant viewing direction (e.g., `greatestDir`) [S1, unofficial].

**Steps:**
1.  **Compute Camera Block Coordinates:** Calculate the integer block coordinates of the camera by flooring its world position [S1, unofficial]:
    $$ B_x = \lfloor P_x / S \rfloor, \quad B_y = \lfloor P_y / S \rfloor, \quad B_z = \lfloor P_z / S \rfloor $$
2.  **Determine Dominant Viewing Direction:** Compute the view vector components $(v_x, v_y, v_z)$ from the camera's rotation angles $(r_x, r_y)$ [S1, unofficial]. Identify the axis with the largest absolute component as the greatest viewing direction $D_{curr}$ [S1, unofficial].
3.  **Check for Culling Condition:** Compare the current camera block coordinates $(B_x, B_y, B_z)$ and greatest viewing direction $D_{curr}$ with the previous frame's values $(B_{prev\_x}, B_{prev\_y}, B_{prev\_z})$ and $D_{prev}$ [S1, unofficial].
    *   If $B_x = B_{prev\_x}$ AND $B_y = B_{prev\_y}$ AND $B_z = B_{prev\_z}$ AND $D_{curr} = D_{prev}$, skip the index update process for this block [S1, unofficial].
4.  **Update State:** If the culling condition is not met, proceed with block rendering or mesh generation. Update the previous frame's state with the current values [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def block_visibility_culling(camera_pos, camera_angle, prev_block_coords, prev_greatest_dir, block_size):
    # Step 1: Compute camera block coordinates
    bx = floor(camera_pos.x / block_size)
    by = floor(camera_pos.y / block_size)
    bz = floor(camera_pos.z / block_size)
    
    # Step 2: Determine dominant viewing direction
    # Assuming camera_angle is in degrees, convert to radians
    theta_rad = camera_angle * PI / 180
    cos_theta = cos(theta_rad)
    sin_theta = sin(theta_rad)
    
    # Compute view vector components (simplified 2D rotation for horizontal angle)
    # vx, vy, vz depend on specific camera rotation model (rx, ry)
    # For this example, assume a simple horizontal angle affects x and y projection
    vx = -sin_theta
    vy = -cos_theta
    vz = 0 # Simplified; real implementation uses rx, ry
    
    # Determine greatest direction (axis with largest absolute component)
    abs_vx = abs(vx)
    abs_vy = abs(vy)
    abs_vz = abs(vz)
    
    if abs_vx >= abs_vy and abs_vx >= abs_vz:
        curr_greatest_dir = 'X'
    elif abs_vy >= abs_vx and abs_vy >= abs_vz:
        curr_greatest_dir = 'Y'
    else:
        curr_greatest_dir = 'Z'
        
    # Step 3: Check for culling condition
    if (bx == prev_block_coords.x and 
        by == prev_block_coords.y and 
        bz == prev_block_coords.z and 
        curr_greatest_dir == prev_greatest_dir):
        return True # Skip update
    else:
        return False # Proceed with update
        
    # Step 4: Update state (handled by caller)
```

**Parameters:**
*   **Block Size ($S$):** Not given in the sources.
*   **Surface Scale Multiplier:** 120, used for floor/ceiling projection tile calculation [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point errors in camera position may cause flickering when the camera is near block boundaries. Using floor() is standard, but consistent coordinate systems are required [S1, unofficial].
*   **Rotation Changes:** If the camera rotates significantly within the same block, the greatest viewing direction may change, triggering an update even if the position hasn't changed [S1, unofficial].
*   **Degenerate Input:** Camera position exactly on a block boundary may lead to ambiguous block assignment. Consistent flooring handles this, but visual artifacts may occur if not managed [S1, unofficial].

## Subtopics

### [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md)
Determines visibility of individual faces within a block based on geometric relationships. Choose this when fine-grained face-level culling is needed within a block, unlike the coarse block-level skip of the parent method.

## Operating characteristics
*   **Time Complexity:** $O(1)$ for the culling check itself, as it involves simple comparisons of integer coordinates and direction flags [S1, unofficial].
*   **Memory Complexity:** $O(1)$ additional memory for storing previous frame's state [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales well with large scenes as it avoids processing entire blocks when the camera is static or moving slowly within a block [S1, unofficial].
*   **Strengths:** Reduces unnecessary mesh updates and rendering calls when the camera is stable [S1, unofficial].
*   **Limitations:** Does not cull faces within a block; only skips the entire block processing [S1, unofficial]. Requires tracking previous frame's state [S1, unofficial].
*   **When to Use:** When the camera moves slowly or remains static for extended periods, and block-level granularity is sufficient [S1, unofficial].
*   **When Not to Use:** When fine-grained face culling is required or when the camera moves rapidly between blocks [S1, unofficial].

## Background
Originating from implementations in voxel-based game engines, this method is documented in the `exolands` project [S1, unofficial] and `Voxel1993` renderer [S2, unofficial]. It leverages the discrete nature of voxel grids to optimize rendering by avoiding redundant calculations when the viewer's perspective relative to the grid does not change significantly [S1, unofficial][S2, unofficial].

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S2] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
