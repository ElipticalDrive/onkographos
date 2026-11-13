# Block Visibility Culling

Also called: Block Coordinate Change Detection, Block Rendering & Surface Orientation Culling, Camera-relative Block Culling, Visibility Culling via Normal Dot Product, renderBlock

## Overview
Block Visibility Culling determines which coarse voxel blocks require mesh regeneration or rendering updates based on the camera's position and rotation, avoiding unnecessary computation when the viewing direction or block occupancy relative to the camera remains stable [S1, unofficial]. It solves the problem of redundant geometry processing in large voxel worlds by caching visibility states and only recalculating surface normals and face visibility when the camera crosses block boundaries or significantly changes orientation [S1, unofficial]. Unlike sibling methods that operate on individual faces or fine-grained voxel data, this method operates at the block level, using camera-relative coordinates to batch-cull entire blocks or their constituent faces [S3, unofficial].

## How it works
**Inputs:**
- Camera world position $(P_x, P_y, P_z)$.
- Camera rotation angles $(\theta_x, \theta_y)$ in degrees or radians [S1, unofficial].
- Block size $B$ (typically 16 or 32 units, depending on implementation) [S3, unofficial].
- Previous frame's camera block coordinates $(C_{x,prev}, C_{y,prev}, C_{z,prev})$ and greatest viewing direction $D_{prev}$ [S1, unofficial].
- Block data structure containing occupancy flags and visible face masks [S3, unofficial].

**Outputs:**
- A set of blocks marked for update or rendering.
- Updated visible face masks for each block [S3, unofficial].

**Data Structures:**
- `BlockFace` enum or bitmask with flags: `YNeg`, `XPos`, `YPos`, `XNeg`, `ZNeg`, `ZPos` [S3, unofficial].
- `visibleFaces` array or bitmask per block, storing which of the 6 faces are visible [S3, unofficial].
- Camera state: `blockX`, `blockY`, `blockZ` (integer coordinates) [S1, unofficial].

**Steps:**

1. **Compute Camera Block Coordinates:**
   Calculate the integer block coordinates containing the camera by flooring the world position divided by block size:
   $$
   C_x = \lfloor P_x / B \rfloor, \quad C_y = \lfloor P_y / B \rfloor, \quad C_z = \lfloor P_z / B \rfloor
   $$
   [S1, unofficial]

2. **Check for Stale State (Optimization):**
   If the current camera block coordinates $(C_x, C_y, C_z)$ and the primary viewing direction $D$ match the previous frame's values $(C_{x,prev}, C_{y,prev}, C_{z,prev}, D_{prev})$, skip the index update process for all blocks [S1, unofficial]. This avoids redundant calculations when the camera has not moved significantly [S1, unofficial].

3. **Compute View Vector:**
   Calculate the view direction components $(v_x, v_y, v_z)$ from the camera's rotation angles $(\theta_x, \theta_y)$ [S1, unofficial].
   $$
   v_x = \cos(\theta_y) \cos(\theta_x)
   $$
   $$
   v_y = \sin(\theta_x)
   $$
   $$
   v_z = \sin(\theta_y) \cos(\theta_x)
   $$
   Normalize the vector if necessary [S2, unofficial].

4. **Determine Block Surface Orientation:**
   For each block, calculate its center or bounds relative to the camera. Determine the surface orientation based on the camera's vertical position relative to the block's vertical bounds ($Z_{bottom}, Z_{top}$) [S3, unofficial]:
   - If $P_z < Z_{bottom}$, orientation is Top (1).
   - If $P_z > Z_{top}$, orientation is Bottom (2).
   - Otherwise, orientation is Side (0) [S3, unofficial].

5. **Calculate Face Visibility via Normal Dot Product:**
   For each face of the block, compute the average normal $N$ (mean of vertex normals, normalized) [S2, unofficial]. Calculate the view direction $V$ from the face center to the camera position and normalize it [S2, unofficial].
   $$
   V = \frac{P - C_{face}}{\|P - C_{face}\|}
   $$
   If the dot product $N \cdot V > 0$, the surface is considered visible [S2, unofficial]. Note: Some implementations use the camera's forward vector directly instead of the vector to the face center, depending on whether the culling is face-centric or block-centric [S1, unofficial].

6. **Transform to Camera Space (Optional for Rendering):**
   For blocks marked visible, transform block coordinates to camera space for rendering:
   $$
   cs = \cos(\theta_y \cdot \pi/180), \quad sn = \sin(\theta_y \cdot \pi/180)
   $$
   $$
   blockX = C_x \cdot B + x \cdot B
   $$
   $$
   blockZ_{bottom} = C_z \cdot B + z \cdot B
   $$
   $$
   blockZ_{top} = blockZ_{bottom} + B
   $$
   Compute bottom corners and project to screen coordinates using the rotation matrix [S3, unofficial].

7. **Update Visible Face Mask:**
   Store the result in `block->visibleFaces[BlockFace::...]` before rendering [S3, unofficial]. If a face is culled, it is not added to the mesh indices [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def block_visibility_culling(camera_pos, camera_angles, block_size, prev_camera_state, blocks):
    """
    Determine which blocks need rendering based on camera position and rotation.
    
    Args:
        camera_pos: (x, y, z) world coordinates of the camera.
        camera_angles: (theta_x, theta_y) rotation angles in degrees.
        block_size: Size of a single voxel block.
        prev_camera_state: (block_x, block_y, block_z, view_dir) from previous frame.
        blocks: List of block objects with occupancy and visibleFaces attributes.
        
    Returns:
        List of blocks that need to be rendered or updated.
    """
    # Step 1: Compute Camera Block Coordinates
    cx = floor(camera_pos.x / block_size)
    cy = floor(camera_pos.y / block_size)
    cz = floor(camera_pos.z / block_size)
    
    # Step 2: Check for Stale State
    current_view_dir = compute_view_direction(camera_angles)
    if (cx, cy, cz) == prev_camera_state[:3] and current_view_dir == prev_camera_state[3]:
        return []  # Skip update, no change in visibility state
    
    # Update previous state for next frame
    prev_camera_state = (cx, cy, cz, current_view_dir)
    
    visible_blocks = []
    
    for block in blocks:
        if not block.occupied:
            continue
            
        # Step 3: Compute View Vector (already done as current_view_dir)
        view_dir = current_view_dir
        
        # Step 4: Determine Block Surface Orientation
        # Calculate block bounds
        block_min_x = cx * block_size
        block_max_x = (cx + 1) * block_size
        block_min_y = cy * block_size
        block_max_y = (cy + 1) * block_size
        block_min_z = cz * block_size
        block_max_z = (cz + 1) * block_size
        
        # Determine orientation relative to camera
        if camera_pos.z < block_min_z:
            orientation = 1  # Top
        elif camera_pos.z > block_max_z:
            orientation = 2  # Bottom
        else:
            orientation = 0  # Side
            
        # Step 5: Calculate Face Visibility
        # For each of the 6 faces, check if visible
        # Simplified: Use dot product of face normal and view direction
        # Face normals: (+-1, 0, 0), (0, +-1, 0), (0, 0, +-1)
        
        faces_to_render = []
        
        # Check X faces
        if dot((1, 0, 0), view_dir) > 0:
            faces_to_render.append('XPos')
        if dot((-1, 0, 0), view_dir) > 0:
            faces_to_render.append('XNeg')
            
        # Check Y faces
        if dot((0, 1, 0), view_dir) > 0:
            faces_to_render.append('YPos')
        if dot((0, -1, 0), view_dir) > 0:
            faces_to_render.append('YNeg')
            
        # Check Z faces
        if dot((0, 0, 1), view_dir) > 0:
            faces_to_render.append('ZPos')
        if dot((0, 0, -1), view_dir) > 0:
            faces_to_render.append('ZNeg')
            
        # Step 6: Update Visible Face Mask
        block.visibleFaces = faces_to_render
        
        # Step 7: Add to render list if any face is visible
        if faces_to_render:
            visible_blocks.append(block)
            
    return visible_blocks

def compute_view_direction(angles_deg):
    """Compute view direction from camera angles."""
    theta_x = radians(angles_deg[0])
    theta_y = radians(angles_deg[1])
    
    # Standard spherical coordinates conversion
    # Note: Exact formula may vary based on coordinate system (right-handed/left-handed)
    # Using common FPS camera model:
    cos_x = cos(theta_x)
    sin_x = sin(theta_x)
    cos_y = cos(theta_y)
    sin_y = sin(theta_y)
    
    # View direction vector
    vx = cos_y * cos_x
    vy = sin_x
    vz = sin_y * cos_x
    
    # Normalize
    length = sqrt(vx**2 + vy**2 + vz**2)
    if length > 0:
        vx /= length
        vy /= length
        vz /= length
        
    return (vx, vy, vz)

def dot(v1, v2):
    """Dot product of two vectors."""
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]
```

**Parameters:**
- `block_size`: Typically 16 or 32 units [S3, unofficial].
- `surfaceScale`: Multiplier for floor/ceiling projection, typically 120 [S3, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point errors in block coordinate calculation can cause flickering when the camera is near block boundaries. Use epsilon comparisons or consistent flooring [S1, unofficial].
- **Degenerate Input:** If the camera is exactly on a block boundary, the orientation calculation may be ambiguous. Ensure consistent handling of boundary cases (e.g., always treat as inside or outside) [S3, unofficial].
- **View Direction Ambiguity:** Using the camera's forward vector vs. the vector to the face center can yield different results. The forward vector is faster but less accurate for large blocks [S1, unofficial].

## Subtopics

### [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md)
Operates at the fine-grained face level within a block, determining visibility by evaluating the geometric relationship between face normals and a reference direction. Choose this when per-face precision is required, unlike the coarse block-level culling of the parent method.

## Operating characteristics
- **Time Complexity:** $O(1)$ per block for the visibility check, assuming a fixed number of faces. The overall complexity is $O(N)$ where $N$ is the number of blocks, but with significant optimization via the stale state check [S1, unofficial].
- **Memory Complexity:** $O(1)$ per block for storing the visible face mask [S3, unofficial].
- **Measured Performance:** Reduces rendering overhead by skipping mesh updates when the camera is stationary or moving within the same block [S1, unofficial]. Performance gains are most significant in large, static scenes [S1, unofficial].
- **Strengths:** Simple to implement, low computational overhead, effective for static or slowly moving cameras [S1, unofficial].
- **Limitations:** Does not account for occlusion by other blocks or geometry, only visibility based on orientation [S2, unofficial]. May cull faces that are technically visible but occluded by other objects [S1, unofficial].
- **When to Use:** Ideal for voxel-based games or applications where the camera moves in discrete steps or within large blocks, and occlusion is handled by other means (e.g., frustum culling) [S1, unofficial].
- **When Not to Use:** In scenes with complex occlusion or when the camera moves rapidly between blocks, as the coarse culling may miss visible faces [S1, unofficial].

## Background
Block Visibility Culling originated in voxel-based rendering engines, particularly those inspired by games like Minecraft, where efficient rendering of large voxel worlds is critical [S1, unofficial]. The method was popularized by implementations that optimized mesh generation by caching visibility states and only updating blocks when the camera's relative position changed [S1, unofficial]. It is commonly used in real-time voxel renderers and game engines that prioritize performance over perfect visual accuracy [S3, unofficial].

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S2] matthxie/NeRF-Renderer, https://github.com/matthxie/NeRF-Renderer, fetched 2026-10-27 (unofficial)
[S3] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
