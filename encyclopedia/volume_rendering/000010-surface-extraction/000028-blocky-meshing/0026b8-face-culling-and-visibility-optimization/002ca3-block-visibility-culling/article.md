# Block Visibility Culling
Also called: Block Coordinate Change Detection, Block Rendering & Surface Orientation Culling, Camera-relative Block Culling, renderBlock

## Overview
Block Visibility Culling determines which voxel blocks require mesh updates or rendering based on the camera's position and rotation, avoiding unnecessary geometry processing when the camera has not moved significantly or changed its primary viewing direction [S1, unofficial]. It operates at the coarse block level by comparing current camera coordinates against previous frame values to skip index updates entirely, or by calculating screen-space orientation to cull invisible faces within a block [S1, unofficial][S2, unofficial]. Unlike [Face Culling](8735-face-culling.md) which checks neighbor occupancy for every voxel, this method optimizes the update loop itself by leveraging camera-relative visibility flags [S2, unofficial].

## How it works
**Inputs:** Camera world position $(P_x, P_y, P_z)$, camera rotation angle $\theta$ (in degrees), block grid coordinates $(x, y, z)$, block size $B$, and previous frame's camera block coordinates $(C_x^{prev}, C_y^{prev}, C_z^{prev})$ and greatest viewing direction $D^{prev}$.
**Outputs:** Boolean flag `skipUpdate` indicating if the block's mesh indices need regeneration, or a set of visible face flags for the block.

**Data Structures:**
*   `BlockFace`: An enum or bitmask with 6 flags: `YNeg` (bottom), `XPos` (right), `YPos` (top), `XNeg` (left), `ZNeg` (back), `ZPos` (front).
*   `visibleFaces`: An array or integer storing the bitmask of visible faces for a block.

**Steps:**
1.  **Compute Camera Block Coordinates:**
    Calculate the integer block coordinates containing the camera:
    $$ C_x = \lfloor P_x / B \rfloor, \quad C_y = \lfloor P_y / B \rfloor, \quad C_z = \lfloor P_z / B \rfloor $$
    [S1, unofficial]

2.  **Check for Stasis (Optimization):**
    Determine the current greatest viewing direction $D$ based on camera rotation angles $(r_x, r_y)$. If the current camera block coordinates $(C_x, C_y, C_z)$ and $D$ match the previous frame's values $(C_x^{prev}, C_y^{prev}, C_z^{prev})$ and $D^{prev}$, skip the index update process for all blocks in the chunk.
    $$ \text{if } (C_x == C_x^{prev} \land C_y == C_y^{prev} \land C_z == C_z^{prev} \land D == D^{prev}) \text{ then return skip} $$
    [S1, unofficial]

3.  **Calculate Surface Orientation:**
    For a specific block at grid position $(x, y, z)$, compute its world-space vertical bounds:
    $$ Z_{bottom} = z \cdot B $$
    $$ Z_{top} = (z + 1) \cdot B $$
    Determine the vertical orientation relative to the camera:
    $$ \text{orientation} = \begin{cases} 1 & \text{if } P_z < Z_{bottom} \quad (\text{Camera below block}) \\ 2 & \text{if } P_z > Z_{top} \quad (\text{Camera above block}) \\ 0 & \text{otherwise} \quad (\text{Camera inside or level with block}) \end{cases} $$
    [S2, unofficial]

4.  **Determine Visible Faces:**
    Based on `orientation` and horizontal position relative to the camera, set the `visibleFaces` flags.
    *   If `orientation` $\neq 0$, the top and bottom faces are potentially visible (cycles = 2).
    *   If `orientation` $== 0$, only side faces are considered (cycles = 1).
    *   The specific side faces (`XPos`, `XNeg`, `YPos`, `YNeg`) are determined by projecting the block's corners into camera space and checking visibility flags.
    $$ \text{cycles} = \begin{cases} 2 & \text{if orientation} \neq 0 \\ 1 & \text{if orientation} == 0 \end{cases} $$
    [S2, unofficial]

5.  **Transform and Project (for face culling):**
    For each side face $w \in [0, 3]$:
    *   Get bottom corner coordinates $(x_{bottom}, y_{bottom})$.
    *   Transform to camera space:
        $$ x_1 = x_{bottom} - P_x $$
        $$ y_1 = y_{bottom} - P_y $$
    *   Apply rotation matrix using $cs = \cos(\theta \cdot \pi/180)$ and $sn = \sin(\theta \cdot \pi/180)$:
        $$ \text{wallX}[w][0] = x_1 \cdot cs - y_1 \cdot sn $$
    *   Swap $x_1/x_2$ and $y_1/y_2$ if `orientation` == 1 (camera below).
    *   Check `block->visibleFaces[BlockFace::...]` before rendering. If the flag is false, skip the face.
    [S1, unofficial][S2, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def block_visibility_culling(camera_pos, camera_angle_deg, block_grid_pos, block_size, prev_camera_block, prev_dir):
    # 1. Compute Camera Block Coordinates
    cx = floor(camera_pos.x / block_size)
    cy = floor(camera_pos.y / block_size)
    cz = floor(camera_pos.z / block_size)
    
    # 2. Check for Stasis
    current_dir = get_greatest_viewing_direction(camera_angle_deg)
    if (cx == prev_camera_block.x and 
        cy == prev_camera_block.y and 
        cz == prev_camera_block.z and 
        current_dir == prev_dir):
        return SKIP_UPDATE

    # 3. Calculate Surface Orientation
    z_bottom = block_grid_pos.z * block_size
    z_top = (block_grid_pos.z + 1) * block_size
    
    if camera_pos.z < z_bottom:
        orientation = 1
    elif camera_pos.z > z_top:
        orientation = 2
    else:
        orientation = 0
        
    cycles = 2 if orientation != 0 else 1
    
    # 4. Determine Visible Faces
    visible_faces = 0
    
    # Check vertical faces based on horizontal position and orientation
    # Note: Exact logic for XPos/XNeg/YPos depends on specific camera-space projection
    # Simplified check based on source [S2, unofficial] logic for side walls
    if orientation == 0:
        # Camera is level/inside, check side visibility
        # This typically involves checking if the face normal points towards the camera
        pass 
        
    # 5. Transform and Project for rendering
    cs = cos(camera_angle_deg * PI / 180)
    sn = sin(camera_angle_deg * PI / 180)
    
    for w in range(4): # 4 side faces
        # Get bottom corners for face w
        x_bottom, y_bottom = get_face_bottom_corners(w, block_grid_pos, block_size)
        
        # Transform to camera space
        x1 = x_bottom - camera_pos.x
        y1 = y_bottom - camera_pos.y
        
        # Apply rotation
        wall_x = x1 * cs - y1 * sn
        
        # Swap if camera is below
        if orientation == 1:
            swap_x_y_coordinates()
            
        # Check visibility flag before adding to mesh
        face_flag = get_face_flag(w, orientation)
        if visible_faces & face_flag:
            add_face_to_mesh(w)
            
    return visible_faces
```

## Subtopics

### [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md)
Determines visibility of individual faces within a block by evaluating geometric relationships between face normals and the view vector. It operates at a finer granularity than Block Visibility Culling, which skips entire block updates or uses coarse orientation flags. Choose this when precise per-face culling is needed within a block that is already determined to be visible.

## Operating characteristics
**Time Complexity:** $O(1)$ per block for the stasis check. $O(N)$ for processing $N$ blocks if stasis is not triggered. The stasis check provides significant performance gains in static scenes or slow-moving cameras.
**Memory Complexity:** $O(1)$ additional memory per block for storing `visibleFaces` flags.
**Performance:** The culling check is performed once during mesh creation/update, resulting in no framerate loss during the rendering phase [S1, unofficial].
**Strengths:** Eliminates redundant mesh generation when the camera is stationary or moving slowly within the same block. Reduces GPU draw calls by culling invisible faces early.
**Limitations:** Requires tracking previous frame state. Less effective in highly dynamic scenes with rapid camera movement. Does not account for occlusion by other blocks (unlike [Face Culling](8735-face-culling.md)).

## Background
Block Visibility Culling is a technique used in voxel-based rendering engines to optimize the update loop. It is implemented in projects such as Exolands [S1, unofficial] and Voxel1993 [S2, unofficial]. The method leverages the fact that voxel blocks are often rendered as simple cubes, and their visibility can be determined by simple geometric tests relative to the camera.

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S2] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
