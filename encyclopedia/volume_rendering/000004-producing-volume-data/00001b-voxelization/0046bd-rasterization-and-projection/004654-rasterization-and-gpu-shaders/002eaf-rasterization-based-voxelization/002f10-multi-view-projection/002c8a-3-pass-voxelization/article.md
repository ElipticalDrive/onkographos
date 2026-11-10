# 3-Pass Voxelization

Also called: 3-Axis Voxelization, 3D Mesh Voxelization via Axis-Aligned Rasterization, Multi-View Voxelization

## Overview
3-Pass Voxelization is a rasterization-based technique that eliminates cracks and missing data in the voxel grid by rendering the scene from three orthogonal directions, one for each principal axis (X, Y, Z) [S1, S2, unofficial]. Unlike single-view projection methods which suffer from occlusion shadows and geometric distortion, this approach ensures that every surface element is visible from at least one viewpoint, guaranteeing complete voxel occupancy [S1, S2, unofficial]. It is chosen when geometric completeness is critical and the computational cost of three render passes per object is acceptable.

## How it works
**Inputs:**
- A 3D mesh (triangle list).
- A scene bounding box defining the voxel grid extent.
- A target voxel grid resolution $(R_x, R_y, R_z)$.
- A 3D texture or array representing the voxel grid.

**Outputs:**
- A populated 3D voxel grid where occupied voxels indicate surface presence.

**Data Structures:**
- **Voxel Grid:** A 3D array or 3D texture of size $R_x \times R_y \times R_z$.
- **Projection Matrices:**
  - $M_{clip}$: The standard clip-space projection matrix.
  - $M_{voxel}$: A matrix mapping clip-space coordinates directly to voxel grid indices.
  - $R_x, R_y, R_z$: Rotation matrices aligning the view with the X, Y, and Z axes respectively.

**Steps:**
1. **Initialize Coordinate System:** Place the coordinate system at the center of the voxel grid, ensuring the axes are parallel to the grid edges [S2, unofficial].
2. **Define View Directions:** Identify the three principal axes: X, Y, and Z [S2, unofficial].
3. **Render Pass Loop:** For each axis $A \in \{X, Y, Z\}$:
   a. **Compute View Matrix:** Multiply the clip-space projection matrix $M_{clip}$ by the rotation matrix $R_A$ corresponding to the current axis to generate fragments from that view direction [S1, unofficial].
   b. **Rasterize:** Execute the rendering pass viewing the scene along the axis direction (e.g., +Z or -Z) [S2, unofficial].
   c. **Map to Voxel Space:** Use the pre-computed voxel-space matrix $M_{voxel}$ to transform fragment positions. Note that $M_{voxel}$ does not need to be multiplied by the rotation matrix $R_A$ because it already maps values properly; the rotation is only required to ensure fragments are generated in the correct pass [S1, unofficial].
   d. **Write to Grid:** Write the fragment data to the corresponding voxel coordinates in the 3D grid. Use atomic operations if writing directly to a shared 3D texture to handle concurrent writes from multiple triangles or passes [G].
4. **Completion:** After all three passes, the voxel grid contains data from all orthogonal views, filling gaps that would exist in a single-pass approach [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def three_pass_voxelization(mesh, bounding_box, resolution):
    # 1. Initialize voxel grid
    voxel_grid = create_3d_array(resolution)
    
    # 2. Define rotation matrices for X, Y, Z axes
    # These align the camera view with the respective principal axis
    rotations = [
        get_rotation_matrix_for_axis('X'),
        get_rotation_matrix_for_axis('Y'),
        get_rotation_matrix_for_axis('Z')
    ]
    
    # 3. Pre-compute voxel-space matrix (maps clip space to voxel indices)
    # This matrix is constant and does not change per axis
    voxel_matrix = compute_voxel_matrix(bounding_box, resolution)
    
    # 4. Standard clip-space projection matrix
    clip_matrix = compute_clip_projection_matrix()
    
    # 5. Execute 3 render passes
    for rotation in rotations:
        # Multiply clip matrix by rotation to get view-specific projection
        view_projection = multiply_matrices(clip_matrix, rotation)
        
        # Rasterize mesh using the view projection
        # This generates fragments visible from this axis
        fragments = rasterize_mesh(mesh, view_projection)
        
        # 6. Map fragments to voxel coordinates
        for frag in fragments:
            # Transform fragment position to voxel space
            # Note: voxel_matrix is used directly, not multiplied by rotation
            voxel_coords = transform_with_matrix(frag.position, voxel_matrix)
            
            # 7. Write to voxel grid
            # Handle potential concurrent writes with atomic operations
            atomic_write(voxel_grid, voxel_coords, frag.data)
            
    return voxel_grid
```

**Parameters:**
- **Resolution:** Determines the granularity. Higher resolution increases accuracy but also memory and compute cost [S1, unofficial].
- **Bounding Box:** Defines the spatial extent of the voxel grid.

**Edge Cases and Pitfalls:**
- **Performance Cost:** Requires 3 render calls per object, which becomes expensive for complex models [S1, unofficial].
- **Dynamic Objects:** If optimizing by separating dynamic and static objects, the voxel grid for dynamic objects must be cleared every frame, necessitating separate voxel grids for dynamic and static content [S1, unofficial].
- **Degeneracy:** Triangles parallel to a view direction may project to a line or point. The multi-view nature helps mitigate this, as a triangle invisible from one axis may be visible from another [G].

## Operating characteristics
- **Time Complexity:** $O(N \cdot V)$, where $N$ is the number of triangles and $V$ is the number of views (3). Each pass requires rasterizing the entire mesh.
- **Memory Complexity:** $O(R_x \cdot R_y \cdot R_z)$ for the voxel grid. If dynamic/static separation is used, memory usage doubles [S1, unofficial].
- **Strengths:** Eliminates cracks and missing data inherent in single-view projection; simple to implement; robust against occlusion from specific angles [S1, S2, unofficial].
- **Limitations:** High computational cost due to multiple render passes; not suitable for real-time applications with very high triangle counts without optimization (e.g., LOD interleaving, resolution scaling) [S1, unofficial].
- **When to use:** When geometric completeness is more important than performance, or when single-view methods produce unacceptable artifacts.
- **When not to use:** When performance is critical and the scene is simple enough that single-view projection suffices, or when the hardware cannot support 3 render passes per frame efficiently.

## Background
3-Pass Voxelization was introduced as a solution to the problem of cracks and missing data in voxel grids generated by single-direction rasterization [S1, unofficial]. It is described in the context of real-time ray-traced voxel global illumination, where accurate surface representation is crucial for lighting calculations [S1, unofficial]. The method leverages the principle that orthogonal projections from three axes ensure full coverage of the 3D object's surface [S2, unofficial].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] 3D-Mesh-Voxelizer - GitHub, https://github.com/Talkative-Banana/3D-Mesh-Voxelizer, fetched 2026-10-26 (unofficial)
