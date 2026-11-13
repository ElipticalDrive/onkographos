# Block Visibility Culling

Also called: Block Coordinate Change Detection, Block Rendering & Surface Orientation Culling, Camera-relative Block Culling, Visibility Culling via Normal Dot Product, renderBlock

## Overview
Block Visibility Culling determines which voxel blocks require mesh updates or rendering based on the camera's position and rotation, avoiding unnecessary geometry generation when the viewing direction or block-relative position remains stable [S1, S3, unofficial]. It operates at the coarse block level by comparing the current camera block coordinates and primary viewing direction against the previous frame's values; if both match, the index update process is skipped entirely [S1, unofficial]. This method complements fine-grained face culling by preventing the overhead of processing blocks that are not facing the viewer or have not changed relative visibility [S3, unofficial].

## How it works
**Inputs:**
- Camera world position $(P_x, P_y, P_z)$.
- Camera rotation angles (yaw $A$, pitch $R_x, R_y$).
- Previous frame's camera block coordinates $(B_{x,prev}, B_{y,prev}, B_{z,prev})$ and greatest direction indicator $G_{prev}$.
- Block size $S$ (typically 16 units in voxel engines [S3, unofficial]).
- Chunk coordinates $(C_x, C_y, C_z)$.

**Outputs:**
- Boolean flag `skip_update`: true if the block mesh does not need regeneration.
- Visibility flags for each of the six block faces (`BlockFace::YNeg`, `XPos`, `YPos`, `XNeg`, `ZNeg`, `ZPos`).

**Data Structures:**
- `BlockFace`: An enumeration or bitmask representing the six faces of a voxel block.
- `visibleFaces`: An array or bitmask associated with each block, storing the visibility state of its faces [S3, unofficial].

**Steps:**

1.  **Compute Camera Block Coordinates:**
    Calculate the integer block coordinates containing the camera by flooring the world position divided by the block size:
    $$
    B_x = \lfloor P_x / S \rfloor, \quad B_y = \lfloor P_y / S \rfloor, \quad B_z = \lfloor P_z / S \rfloor
    $$
    [S1, unofficial]

2.  **Check for Stasis (Optimization):**
    Determine the current primary viewing direction (greatest direction) $G_{curr}$ from the camera rotation angles. If the current block coordinates $(B_x, B_y, B_z)$ match the previous frame's coordinates $(B_{x,prev}, B_{y,prev}, B_{z,prev})$ AND the greatest direction $G_{curr}$ matches $G_{prev}$, set `skip_update = true` and terminate the culling process for this block [S1, unofficial].

3.  **Calculate View Vector:**
    Compute the view vector components $(v_x, v_y, v_z)$ from the camera's rotation angles (e.g., $R_x, R_y$) [S1, unofficial]. Alternatively, compute the view direction from the block center to the camera position and normalize it [S2, unofficial].

4.  **Determine Surface Orientation and Visibility:**
    For each of the six faces of the block:
    a.  **Calculate Face Center and Normal:**
        Identify the vertices of the face. Calculate the center $C_f$ as the mean of its vertices. Calculate the average normal $\mathbf{n}$ as the mean of its vertex normals, then normalize it [S2, unofficial].
    b.  **Compute Dot Product:**
        Calculate the dot product between the face normal $\mathbf{n}$ and the view direction $\mathbf{v}$:
        $$
        d = \mathbf{n} \cdot \mathbf{v}
        $$
    c.  **Evaluate Visibility:**
        If $d > 0$, the surface is considered visible to the camera. Set the corresponding bit in `visibleFaces` to 1. Otherwise, set it to 0 [S2, unofficial].

    *Alternative Orientation Check (for specific engine implementations):*
    Calculate the block's vertical bounds relative to the player:
    $$
    Z_{bottom} = C_z \cdot S, \quad Z_{top} = Z_{bottom} + S
    $$
    Determine `surfaceOrientation`:
    - 1 if $P_z < Z_{bottom}$ (player is below, looking up).
    - 2 if $P_z > Z_{top}$ (player is above, looking down).
    - 0 otherwise [S3, unofficial].
    Set `cycles = 2` if `surfaceOrientation != 0`, else `cycles = 1` [S3, unofficial].

5.  **Transform and Project (Optional for Immediate Mode):**
    If rendering immediately, transform block corners to camera space:
    $$
    x_1 = x_{bottom1}[w] - P_x, \quad y_1 = y_{bottom1}[w] - P_y
    $$
    Swap $x_1/x_2$ and $y_1/y_2$ if `surfaceOrientation == 1` [S3, unofficial].
    Project using camera angle $A$:
    $$
    cs = \cos(A \cdot \pi / 180), \quad sn = \sin(A \cdot \pi / 180)
    $$
    $$
    wallX[w][0] = x_1 \cdot cs - y_1 \cdot sn
    $$
    [S3, unofficial]

6.  **Update State:**
    Store $(B_x, B_y, B_z)$ and $G_{curr}$ as the previous frame's values for the next iteration [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def block_visibility_culling(camera_pos, camera_angles, prev_block_coords, prev_greatest_dir, block_size, block_faces):
    """
    Determine if a block needs mesh update based on camera movement and orientation.
    
    Args:
        camera_pos: Tuple (x, y, z) of camera world position.
        camera_angles: Tuple (yaw, pitch_x, pitch_y) of camera rotation.
        prev_block_coords: Tuple (bx, by, bz) of previous frame's block coords.
        prev_greatest_dir: Integer indicating previous primary view direction.
        block_size: Integer size of a voxel block.
        block_faces: List of face objects, each with 'vertices' and 'normals'.
        
    Returns:
        skip_update: Boolean.
        visible_faces: Dictionary mapping face index to boolean visibility.
    """
    # Step 1: Compute Camera Block Coordinates
    bx = floor(camera_pos.x / block_size)
    by = floor(camera_pos.y / block_size)
    bz = floor(camera_pos.z / block_size)
    
    # Step 2: Check for Stasis
    current_greatest_dir = get_greatest_view_direction(camera_angles)
    
    if (bx == prev_block_coords.x and 
        by == prev_block_coords.y and 
        bz == prev_block_coords.z and 
        current_greatest_dir == prev_greatest_dir):
        return True, {} # Skip update
    
    # Step 3: Calculate View Vector
    view_dir = compute_view_vector(camera_angles)
    
    visible_faces = {}
    
    # Step 4: Determine Surface Orientation and Visibility
    for face_idx, face in enumerate(block_faces):
        # Calculate average normal
        avg_normal = normalize(mean(face.normals))
        
        # Compute dot product
        dot_product = dot(avg_normal, view_dir)
        
        # Evaluate visibility
        if dot_product > 0:
            visible_faces[face_idx] = True
        else:
            visible_faces[face_idx] = False
            
    # Step 6: Update State (returned for next frame)
    new_block_coords = (bx, by, bz)
    new_greatest_dir = current_greatest_dir
    
    return False, visible_faces, new_block_coords, new_greatest_dir

def get_greatest_view_direction(angles):
    """Helper to determine primary viewing axis/direction."""
    # Implementation depends on specific angle representation (e.g., Euler vs Quaternion)
    # Simplified: return index of max absolute component of view vector
    pass

def compute_view_vector(angles):
    """Helper to compute view vector from angles."""
    # Simplified: return normalized vector based on yaw/pitch
    pass
```

**Parameters:**
- `block_size`: Typically 16 units in voxel engines [S3, unofficial].
- `surfaceScale`: Multiplier for floor/ceiling projection, typically 120 [S3, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point errors in dot product calculations near the boundary ($d=0$) may cause flickering. A small epsilon threshold may be required [G].
- **Degenerate Input:** If the camera is exactly on a block boundary, block coordinate calculation must be consistent (e.g., always floor) to avoid jumping between blocks [S1, unofficial].
- **Orientation Ambiguity:** When `surfaceOrientation` is 0, the logic for `cycles` and corner swapping must be handled carefully to avoid rendering artifacts [S3, unofficial].

## Subtopics

### [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md)
Determines visibility of individual faces within a block by evaluating the geometric relationship between face normals and the view direction. It operates at the fine-grained face level, whereas Block Visibility Culling operates at the coarse block level based on camera movement. Choose this when fine-grained optimization is needed within a block that is already determined to be visible.

## Operating characteristics
- **Time Complexity:** $O(1)$ per block for the stasis check. $O(F)$ for face visibility calculation, where $F$ is the number of faces (typically 6).
- **Memory Complexity:** $O(1)$ per block to store `visibleFaces` and previous state.
- **Measured Performance:** Significantly reduces triangle count by avoiding mesh generation for blocks that are not facing the camera or have not changed relative position [S1, S3, unofficial].
- **Strengths:** Low computational overhead; effective for static scenes or slow camera movement.
- **Limitations:** Does not cull faces based on occlusion or distance; relies on camera movement for optimization.
- **When to Use:** Ideal for voxel engines where block mesh regeneration is expensive and camera movement is discrete or slow.
- **When Not to Use:** In highly dynamic scenes with rapid camera movement where the stasis check rarely triggers, or when occlusion culling is more critical.

## Background
Block Visibility Culling originated in voxel-based game engines, particularly those inspired by Minecraft, where efficient rendering of large voxel worlds is critical. The method was implemented in projects like `exolands` [S1, unofficial] and `Voxel1993` [S3, unofficial] to optimize rendering by skipping mesh updates when the camera's relative position and orientation to a block remain unchanged. It leverages the discrete nature of voxel grids to simplify visibility checks to coordinate comparisons and dot products.

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S2] matthxie/NeRF-Renderer, https://github.com/matthxie/NeRF-Renderer, fetched 2026-10-27 (unofficial)
[S3] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
