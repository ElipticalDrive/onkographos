# 3-Pass Voxelization

Also called: 3-Axis Voxelization, 3D Mesh Voxelization via Axis-Aligned Rasterization, Multi-View Voxelization

## Overview
3-Pass Voxelization is a rasterization-based technique that eliminates geometric cracks and missing data in the voxel grid by rendering the scene from three orthogonal directions (along the X, Y, and Z axes) [S1, unofficial][S2, unofficial]. Unlike single-view projection methods which suffer from occlusion gaps and perspective distortion, this method ensures complete volumetric coverage by aggregating fragments from all three principal axes [S2, unofficial]. It is a simple and easy-to-implement approach that trades three draw calls per object for robust occupancy [S1, unofficial].

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
- **Frustum Expansion:** The orthographic frustum should be expanded slightly to ensure corners of the AABB are covered [G].
- **Depth Testing:** Disabled or configured for write-through to capture all intersecting fragments, not just the nearest surface [G].

**Steps:**
1. **Coordinate System Setup:** Place the coordinate system at the center of the voxel grid, ensuring the axes are parallel to the grid edges [S2, unofficial].
2. **Pass 1 (Axis 1):** Construct the view matrix for the first axis (e.g., +Z or -Z) [S2, unofficial]. Multiply the clip-space projection matrix $M_{clip}$ by the rotation matrix corresponding to this axis to generate fragments [S1, unofficial]. Render the scene into a 2D buffer or directly into the voxel grid layer corresponding to this view [S2, unofficial].
3. **Pass 2 (Axis 2):** Construct the view matrix for the second axis (e.g., +X or -X) [S2, unofficial]. Multiply $M_{clip}$ by the rotation matrix for this axis [S1, unofficial]. Render the scene [S2, unofficial].
4. **Pass 3 (Axis 3):** Construct the view matrix for the third axis (e.g., +Y or -Y) [S2, unofficial]. Multiply $M_{clip}$ by the rotation matrix for this axis [S1, unofficial]. Render the scene [S2, unofficial].
5. **Aggregation:** For each voxel hit by any of the three passes, mark it as occupied. This is typically an atomic "OR" operation for occupancy [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxelize_3_pass(mesh, bb_min, bb_max, resolution, voxel_grid):
    # 1. Setup coordinate system at grid center, axes parallel to grid edges [S2, unofficial]
    # 2. Define rotation matrices for the three principal axes [G]
    rot_z = [[1,0,0,0],[0,1,0,0],[0,0,1,0],[0,0,0,1]]
    rot_x = [[1,0,0,0],[0,0,-1,0],[0,1,0,0],[0,0,0,1]]
    rot_y = [[0,0,1,0],[0,1,0,0],[-1,0,0,0],[0,0,0,1]]
    rotations = [rot_z, rot_x, rot_y]

    for rot in rotations:
        # 3. Multiply clip-space projection matrix by rotation to align view [S1, unofficial]
        M_view_proj = matrix_multiply(rot, orthographic_projection_matrix())

        # 4. Rasterize mesh to voxel grid
        for triangle in mesh.triangles:
            # Project vertices to clip space
            p0_clip = matrix_multiply(M_view_proj, triangle.v0)
            p1_clip = matrix_multiply(M_view_proj, triangle.v1)
            p2_clip = matrix_multiply(M_view_proj, triangle.v2)

            # Standard triangle rasterization generates fragments at clip-space positions [G]
            fragments = rasterize_triangle(p0_clip, p1_clip, p2_clip)

            for frag in fragments:
                # 5. Map clip-space coordinates to voxel indices [Parent]
                scale_x = resolution[0] / (bb_max[0] - bb_min[0])
                scale_y = resolution[1] / (bb_max[1] - bb_min[1])
                scale_z = resolution[2] / (bb_max[2] - bb_min[2])
                voxel_index = [
                    floor((frag.p_clip[0] - bb_min[0]) * scale_x),
                    floor((frag.p_clip[1] - bb_min[1]) * scale_y),
                    floor((frag.p_clip[2] - bb_min[2]) * scale_z)
                ]
                # 6. Atomic OR for occupancy [G]
                atomic_or(voxel_grid[voxel_index], 1)

def orthographic_projection_matrix():
    return [[1,0,0,0],[0,1,0,0],[0,0,1,0],[0,0,0,1]] # Identity for standard orthographic view [G]

def matrix_multiply(A, B):
    return [[sum(A[i][k]*B[k][j] for k in range(4)) for j in range(4)] for i in range(4)]

def rasterize_triangle(p0, p1, p2):
    # Standard rasterization logic to generate fragments from clip-space vertices [G]
    return []
```

**Edge Cases and Pitfalls:**
- **Performance:** Requires 3 render calls per object. Cost grows with mesh complexity [S1, unofficial].
- **Dynamic Objects:** If separating dynamic and static objects, dynamic voxel grids must be cleared every frame, requiring separate grids [S1, unofficial].
- **Precision:** Ensure the voxel-space matrix correctly handles the coordinate mapping to avoid off-by-one errors at boundaries [G].

## Operating characteristics
- **Time Complexity:** $O(N \cdot P)$, where $N$ is the number of triangles and $P=3$ is the number of passes. Each pass is a standard rasterization operation [G].
- **Memory Complexity:** $O(R_x \cdot R_y \cdot R_z)$ for the voxel grid. No additional large buffers if rendering directly to the grid [G].
- **Performance:** Slower than single-pass methods due to 3x draw calls. Suitable for static or infrequently updated geometry [S1, unofficial]. Can be optimized by LOD interleaving or resolution scaling [S1, unofficial].
- **Strengths:** Eliminates cracks and gaps inherent in single-view projection; robust against orientation [S2, unofficial].
- **Limitations:** High computational cost for complex dynamic scenes; requires separate handling for dynamic/static splits [S1, unofficial].

## Background
3-Pass Voxelization was introduced as a solution to the "cracks and missing data" problem in single-direction voxelization [S1, unofficial]. It differs from standard perspective or orthographic projection rasterization by using three distinct axis-aligned views to ensure accurate voxel occupancy, avoiding the geometric distortion or gaps inherent in single-view projection [S2, unofficial]. The method is widely used in GPU-based voxelizers where multiple render targets or atomic operations are available [G].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] 3D-Mesh-Voxelizer - GitHub, https://github.com/Talkative-Banana/3D-Mesh-Voxelizer, fetched 2026-10-26 (unofficial)
