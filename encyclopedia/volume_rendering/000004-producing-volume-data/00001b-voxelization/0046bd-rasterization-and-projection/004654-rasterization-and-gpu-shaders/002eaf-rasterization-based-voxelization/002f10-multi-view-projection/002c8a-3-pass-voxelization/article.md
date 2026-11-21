# 3-Pass Voxelization

Also called: 3-Axis Voxelization, 3D Mesh Voxelization via Axis-Aligned Rasterization, Multi-View Voxelization

## Overview
3-Pass Voxelization is a rasterization-based technique that eliminates geometric cracks and missing data in the voxel grid by rendering the scene from three orthogonal directions (along the X, Y, and Z axes) [S1, S2, unofficial]. Unlike single-view projection methods which suffer from occlusion gaps and perspective distortion, this method ensures complete volumetric coverage by aggregating fragments from all three principal axes [S2, unofficial]. It is a simple and easy-to-implement approach that trades three draw calls per object for robust occupancy [S1, unofficial].

## How it works
**Inputs:**
- Triangle mesh geometry (vertices, normals, attributes) [G].
- Scene Axis-Aligned Bounding Box (AABB) defined by $bb_{min}$ and $bb_{max}$ [G].
- Target voxel grid resolution $R = (R_x, R_y, R_z)$ [G].
- Voxel grid data structure (e.g., 3D texture or atomic buffer) [G].

**Outputs:**
- A populated 3D voxel grid where occupied voxels indicate the presence of geometry [G].

**Data Structures:**
- **Voxel Grid:** A 3D array or texture of size $R_x \times R_y \times R_z$. Each voxel stores occupancy (bit/bool) or attributes (floats) [G].
- **Projection Matrices:**
  - $M_{clip}$: Standard clip-space orthographic projection matrix [G].
  - $M_{voxel}$: A matrix mapping clip-space coordinates directly to voxel indices. As noted in [S1, unofficial], this matrix already handles the scaling and offset required for voxel indexing and does *not* require rotation multiplication.
  - $R_x, R_y, R_z$: Rotation matrices aligning the camera with the respective axes [G].

**Parameters:**
- **Frustum Expansion:** The orthographic frustum should be expanded slightly to ensure corners of the AABB are covered [Parent].
- **Depth Testing:** Disabled or configured for write-through to capture all intersecting fragments, not just the nearest surface [Parent].

**Steps:**
1. **Coordinate System Setup:** Place the coordinate system at the center of the voxel grid, ensuring the axes are parallel to the grid edges [S2, unofficial].
2. **Pass 1 (Axis 1):** Construct the view matrix for the first axis (e.g., +Z or -Z) [S2, unofficial]. Multiply the clip-space projection matrix $M_{clip}$ by the rotation matrix corresponding to this axis to generate fragments [S1, unofficial]. Render the scene into a 2D buffer or directly into the voxel grid layer corresponding to this view [S2, unofficial].
3. **Pass 2 (Axis 2):** Construct the view matrix for the second axis (e.g., +X or -X) [S2, unofficial]. Multiply $M_{clip}$ by the rotation matrix for this axis [S1, unofficial]. Render the scene [S2, unofficial].
4. **Pass 3 (Axis 3):** Construct the view matrix for the third axis (e.g., +Y or -Y) [S2, unofficial]. Multiply $M_{clip}$ by the rotation matrix for this axis [S1, unofficial]. Render the scene [S2, unofficial].
5. **Aggregation:** For each voxel hit by any of the three passes, mark it as occupied. This is typically an atomic "OR" operation for occupancy [Parent, G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxelize_3_pass(mesh, bb_min, bb_max, resolution, voxel_grid):
    # 1. Compute voxel-space transformation matrix (no rotation) [S1, unofficial]
    scale = [resolution[0] / (bb_max[0] - bb_min[0]),
             resolution[1] / (bb_max[1] - bb_min[1]),
             resolution[2] / (bb_max[2] - bb_min[2])]
    offset = [-bb_min[0] * scale[0], -bb_min[1] * scale[1], -bb_min[2] * scale[2]]
    M_voxel = scale_matrix(scale) @ translate_matrix(offset)

    # 2. Define rotation matrices for the three principal axes [G]
    rot_z = identity_matrix()
    rot_x = rotation_matrix_x(90.0)
    rot_y = rotation_matrix_y(-90.0)
    rotations = [rot_z, rot_x, rot_y]

    for rot in rotations:
        # 3. Multiply clip-space projection matrix by rotation to align view [S1, unofficial]
        M_view_proj = rot @ M_clip # M_clip is standard orthographic projection [G]
        
        # 4. Rasterize mesh to voxel grid
        # For each triangle in mesh:
        #   Project vertices: p_clip = M_view_proj * p_world
        #   Clip and scan-convert to fragments
        #   For each fragment at clip position p_clip:
        #     i = floor(M_voxel * p_clip.xyz)
        #     atomic_or(voxel_grid[i], 1) # Depth testing disabled to capture all intersections [Parent]

def scale_matrix(s):
    return [[s[0], 0, 0, 0], [0, s[1], 0, 0], [0, 0, s[2], 0], [0, 0, 0, 1]]

def translate_matrix(t):
    return [[1, 0, 0, t[0]], [0, 1, 0, t[1]], [0, 0, 1, t[2]], [0, 0, 0, 1]]

def identity_matrix():
    return [[1, 0, 0, 0], [0, 1, 0, 0], [0, 0, 1, 0], [0, 0, 0, 1]]

def rotation_matrix_x(angle_deg):
    c, s = cos(radians(angle_deg)), sin(radians(angle_deg))
    return [[1, 0, 0, 0], [0, c, -s, 0], [0, s, c, 0], [0, 0, 0, 1]]

def rotation_matrix_y(angle_deg):
    c, s = cos(radians(angle_deg)), sin(radians(angle_deg))
    return [[c, 0, s, 0], [0, 1, 0, 0], [-s, 0, c, 0], [0, 0, 0, 1]]
```

**Edge Cases and Pitfalls:**
- **Performance:** Requires 3 render calls per object. Cost grows with mesh complexity [S1, unofficial].
- **Dynamic Objects:** If separating dynamic and static objects, dynamic voxel grids must be cleared every frame, requiring separate grids [S1, unofficial].
- **Precision:** Ensure the voxel-space matrix correctly handles the coordinate mapping to avoid off-by-one errors at boundaries [Parent].

## Operating characteristics
- **Time Complexity:** $O(N \cdot P)$, where $N$ is the number of triangles and $P=3$ is the number of passes. Each pass is a standard rasterization operation [G].
- **Memory Complexity:** $O(R_x \cdot R_y \cdot R_z)$ for the voxel grid. No additional large buffers if rendering directly to the grid [G].
- **Performance:** Slower than single-pass methods due to 3x draw calls. Suitable for static or infrequently updated geometry. Can be optimized by LOD interleaving or resolution scaling [S1, unofficial].
- **Strengths:** Eliminates cracks and gaps inherent in single-view projection; robust against orientation [S2, unofficial].
- **Limitations:** High computational cost for complex dynamic scenes; requires separate handling for dynamic/static splits [S1, unofficial].

## Background
3-Pass Voxelization was introduced as a solution to the "cracks and missing data" problem in single-direction voxelization [S1, unofficial]. It differs from standard perspective or orthographic projection rasterization by using three distinct axis-aligned views to ensure accurate voxel occupancy, avoiding the geometric distortion or gaps inherent in single-view projection [S2, unofficial]. The method is widely used in GPU-based voxelizers where multiple render targets or atomic operations are available [G].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] 3D-Mesh-Voxelizer - GitHub, https://github.com/Talkative-Banana/3D-Mesh-Voxelizer, fetched 2026-10-26 (unofficial)
