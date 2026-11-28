# Block Visibility Culling

Also called: Block Coordinate Change Detection, Block Rendering & Surface Orientation Culling, Camera-relative Block Culling, Visibility Culling via Normal Dot Product, renderBlock

## Overview
Block Visibility Culling determines which voxel blocks or surface patches need to be rendered based on camera position and orientation, avoiding unnecessary mesh updates when the camera has not moved significantly or changed its primary viewing direction [S1, unofficial]. It solves the problem of rendering invisible surfaces by evaluating the geometric relationship between face normals and the view direction [S2, unofficial]. Unlike [Voxel Block Placement with Face Culling](923-voxel-block-placement-with-face-culling.md) which relies on adjacency to empty space, this method explicitly checks visibility relative to the camera [S1, S3, unofficial]. It differs from [Opacity-based Mesh Culling](8759-opacity-based-mesh-culling.md) by using geometric dot products rather than alpha thresholds [S2, unofficial].

## How it works
**Inputs:** Camera world position $(P_x, P_y, P_z)$, camera rotation angles (typically yaw $A$ in degrees, and optionally pitch/roll), and a set of voxel blocks or surface patches defined by their vertex coordinates and normals [S1, S2, S3, unofficial].
**Outputs:** A boolean mask or list of visible faces/blocks for the current frame [S3, unofficial].

**Data Structures:**
*   `BlockFace`: An enumeration or bitmask representing the six faces of a voxel: `YNeg` (bottom), `XPos` (right), `YPos` (top), `XNeg` (left), `ZNeg` (back), `ZPos` (front) [S3, unofficial].
*   `visibleFaces`: An array or flag set associated with each block, storing which of the six faces are currently visible [S3, unofficial].
*   `previousCameraState`: Stores the integer block coordinates of the camera and the "greatest direction" (primary viewing axis) from the previous frame [S1, unofficial].

**Steps:**

1.  **Compute Camera Block Coordinates:**
    Calculate the integer block coordinates containing the camera by flooring its world position:
    $$
    C_x = \lfloor P_x \rfloor, \quad C_y = \lfloor P_y \rfloor, \quad C_z = \lfloor P_z \rfloor
    $$
    [S1, unofficial]

2.  **Optimization: Skip if Unchanged:**
    Compare the current camera block coordinates $(C_x, C_y, C_z)$ and the primary viewing direction with `previousCameraState`. If both match, skip the index update process for this frame [S1, unofficial].

3.  **Calculate View Direction:**
    Compute the view vector components $(v_x, v_y, v_z)$ from the camera's rotation angles. For a 2D yaw angle $A$ (in degrees):
    $$
    c_s = \cos\left(\frac{A \pi}{180}\right), \quad s_n = \sin\left(\frac{A \pi}{180}\right)
    $$
    The view direction in the horizontal plane is derived from these trigonometric components [S1, S3, unofficial].

4.  **Determine Surface Orientation (Block-Level):**
    For each block, determine its vertical orientation relative to the camera:
    *   Calculate block vertical bounds: $Z_{bottom} = Z_{chunk} + z \cdot B$, $Z_{top} = Z_{bottom} + B$, where $B$ is block size [S3, unofficial].
    *   If $P_z < Z_{bottom}$, orientation is Top (1).
    *   If $P_z > Z_{top}$, orientation is Bottom (2).
    *   Otherwise, orientation is Side (0) [S3, unofficial].
    *   Set cycle count: 2 if orientation != 0, else 1 [S3, unofficial].

5.  **Compute Average Normal:**
    For each surface or face, calculate the average normal $\mathbf{n}$ as the mean of its vertex normals, then normalize it:
    $$
    \mathbf{n} = \frac{\sum_{i=1}^{k} \mathbf{n}_i}{\|\sum_{i=1}^{k} \mathbf{n}_i\|}
    $$
    [S2, unofficial]

6.  **Visibility Check via Dot Product:**
    Compute the view direction vector $\mathbf{v}$ from the surface center to the camera position and normalize it:
    $$
    \mathbf{v} = \frac{P - C_{center}}{\|P - C_{center}\|}
    $$
    [S2, unofficial]
    Calculate the dot product $D = \mathbf{n} \cdot \mathbf{v}$.
    *   If $D > 0$, the surface is visible [S2, unofficial].
    *   If $D \le 0$, the surface is facing away and is culled [S2, unofficial].

7.  **Apply Face Flags (Implementation Specific):**
    In block-based rendering, visibility flags are updated based on the dot product and orientation. These flags are checked before rendering: `if (!block->visibleFaces[BlockFace::YNeg]) continue;` [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_block_visibility(camera_pos, camera_yaw_deg, blocks):
    # 1. Compute camera block coordinates
    cam_block_x = floor(camera_pos.x)
    cam_block_y = floor(camera_pos.y)
    cam_block_z = floor(camera_pos.z)

    # 2. Check for state change (optimization)
    if (cam_block_x == prev_cam_block_x and 
        cam_block_y == prev_cam_block_y and 
        cam_block_z == prev_cam_block_z and 
        camera_yaw_deg == prev_camera_yaw_deg):
        return # Skip index update process

    # 3. Calculate view direction components
    angle_rad = camera_yaw_deg * 3.141592653589793 / 180.0
    cos_a = cos(angle_rad)
    sin_a = sin(angle_rad)
    
    # View direction vector (assuming standard right-handed system where camera looks down -Z or similar)
    # Adjust based on specific coordinate system; here we assume 2D yaw for horizontal visibility
    view_dir_x = -sin_a
    view_dir_z = cos_a
    view_dir_y = 0.0 # Simplified for 2D yaw; use pitch for full 3D

    for block in blocks:
        for face_idx in range(6):
            # Define face normals for the 6 axes
            face_normals = [
                (0, -1, 0), (1, 0, 0), (0, 1, 0), (-1, 0, 0), (0, 0, -1), (0, 0, 1)
            ]
            face_normal = face_normals[face_idx]
            face_center = (block.min + block.max) / 2.0
            
            # 5. Compute vector from face to camera
            vec_to_cam_x = camera_pos.x - face_center.x
            vec_to_cam_y = camera_pos.y - face_center.y
            vec_to_cam_z = camera_pos.z - face_center.z
            
            # Normalize view vector
            length = sqrt(vec_to_cam_x**2 + vec_to_cam_y**2 + vec_to_cam_z**2)
            if length == 0:
                length = 1e-9 # Guard against division by zero [G]
            norm_vec_x = vec_to_cam_x / length
            norm_vec_y = vec_to_cam_y / length
            norm_vec_z = vec_to_cam_z / length
            
            # 6. Dot product test
            dot_product = (face_normal[0] * norm_vec_x + 
                           face_normal[1] * norm_vec_y + 
                           face_normal[2] * norm_vec_z)
            
            # 7. Set visibility flag
            if dot_product > 0:
                block.visible_faces[face_idx] = True
            else:
                block.visible_faces[face_idx] = False

    # Update previous state
    prev_cam_block_x = cam_block_x
    prev_cam_block_y = cam_block_y
    prev_cam_block_z = cam_block_z
    prev_camera_yaw_deg = camera_yaw_deg
```

**Parameters:**
*   `surfaceScale`: Multiplier for floor/ceiling projection calculations, typically 120 [S3, unofficial].
*   `BLOCK_SIZE`: Size of a voxel block, typically 16 units in some implementations [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Floating point errors in dot product near 0 can cause flickering. A small epsilon threshold may be needed, though the sources strictly use $> 0$ [S2, unofficial].
*   **Camera Inside Block:** If the camera is inside a block, the view vector direction may be ambiguous for all faces. The orientation check (Step 4) helps distinguish top/bottom vs sides [S1, unofficial].
*   **Degenerate Input:** If the camera position equals the face center, the view vector is zero-length. A guard value (e.g., $10^{-9}$) should be used for normalization [G].

## Subtopics

### [Voxel Face Visibility Check](6723-voxel-face-visibility-check.md)
Determines which of the six faces of a voxel block are visible by evaluating the geometric relationship between face normals and a reference direction. It solves the problem of rendering unnecessary geometry that faces away from the camera. Choose this when you need per-face visibility flags for block-based rendering.

## Operating characteristics
Not given in the sources.

## Background
Block Visibility Culling is commonly used in voxel-based game engines and terrain rendering systems. The technique leverages the fact that in many voxel scenarios, the majority of faces are internal or back-facing relative to the viewer, allowing for significant performance improvements by skipping their generation or rendering [S1, S3, unofficial].

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S2] matthxie/NeRF-Renderer, https://github.com/matthxie/NeRF-Renderer, fetched 2026-10-27 (unofficial)
[S3] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
