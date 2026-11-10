# 3-Pass Voxelization

Also called: 3-Axis Voxelization, 3D Mesh Voxelization via Axis-Aligned Rasterization, Multi-View Voxelization

## Overview
3-Pass Voxelization is a rasterization-based technique that eliminates cracks and missing data in the voxel grid by rendering the scene from three orthogonal directions (along the X, Y, and Z axes) [S1, S2, unofficial]. Unlike single-view projection methods which suffer from occlusion shadows and geometric distortion, this approach ensures every surface element is visible from at least one viewpoint, guaranteeing accurate voxel occupancy [S2, unofficial]. It is chosen when geometric completeness is prioritized over the computational cost of multiple render passes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D mesh (vertices and indices), a target voxel grid resolution $(N_x, N_y, N_z)$, and the scene's bounding box.
*   **Output:** A populated 3D voxel grid (or 3D texture) representing the scene geometry.

**Data Structures:**
*   **Voxel Grid:** A 3D array or texture of size $N_x \times N_y \times N_z$. Each voxel stores occupancy data (e.g., binary occupancy, signed distance, or normal).
*   **Transformation Matrices:**
    *   $M_{clip}$: The standard clip-space projection matrix (orthographic).
    *   $M_{voxel}$: A matrix mapping world/eye coordinates directly to voxel grid indices.
    *   $R_x, R_y, R_z$: Rotation matrices aligning the view direction with the principal axes.

**Numbered Steps:**

1.  **Initialize Coordinate System:** Place the coordinate system at the center of the voxel grid, ensuring the axes are parallel to the grid edges [S2, unofficial]. Define the bounding box of the scene in world space.

2.  **Define View Matrices:** For each of the three major axes, define a view direction. The view direction is parallel to the axis being projected onto. For example:
    *   Pass 1: View along the Z-axis (projecting onto the XY plane).
    *   Pass 2: View along the X-axis (projecting onto the YZ plane).
    *   Pass 3: View along the Y-axis (projecting onto the XZ plane) [S2, unofficial].

3.  **Construct Projection Matrices:**
    *   For each pass $p \in \{1, 2, 3\}$, calculate the model-view-projection matrix $M_p$.
    *   The sources specify that the clip-space projection matrix $M_{clip}$ must be multiplied by a rotation matrix $R_p$ corresponding to the axis of the current pass to generate fragments from that view-axis [S1, unofficial].
    *   $M_p = M_{clip} \times R_p$ [S1, unofficial].
    *   The rotation is required to ensure that fragments not generated in one pass are generated in at least one of the other passes, covering cracks [S1, unofficial].

4.  **Voxel-Space Mapping:**
    *   The voxel-space matrix $M_{voxel}$ maps coordinates to voxel indices.
    *   The sources state that $M_{voxel}$ does *not* need to be multiplied by the rotation matrix because it already maps all values properly; the rotation is only required for the clip-space generation to ensure correct fragment coverage [S1, unofficial].

5.  **Execute Rendering Passes:**
    *   **Pass 1:** Execute the first rendering pass viewing the scene along one axis direction (e.g., +Z or -Z) to populate the corresponding voxel layer or depth buffer [S2, unofficial].
    *   **Pass 2:** Execute the second rendering pass viewing the scene along a second axis direction (e.g., +X or -X) [S2, unofficial].
    *   **Pass 3:** Execute the third rendering pass viewing the scene along the third axis direction (e.g., +Y or -Y) [S2, unofficial].

6.  **Fragment-to-Voxel Mapping:**
    *   During rasterization, transform fragment coordinates from clip space to voxel grid space using $M_{voxel}$.
    *   Determine the integer voxel coordinates $(i, j, k)$ based on the fragment's position relative to the scene's bounding box and target resolution.
    *   Write to the voxel grid. If multiple triangles write to the same voxel, use atomic operations or a synthesis step to handle concurrent writes [S1, S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxelize_3_pass(mesh, grid_resolution, bounding_box):
    """
    Voxelize a mesh using 3 orthogonal passes.
    
    Args:
        mesh: List of vertices and indices.
        grid_resolution: Tuple (Nx, Ny, Nz) defining the voxel grid dimensions.
        bounding_box: Tuple (min_x, min_y, min_z, max_x, max_y, max_z) defining the scene bounds.
        
    Returns:
        voxel_grid: 3D array of shape (Nx, Ny, Nz).
    """
    Nx, Ny, Nz = grid_resolution
    voxel_grid = create_3d_array(Nx, Ny, Nz)
    
    # Define rotation matrices for each axis
    # R_z rotates around Z axis (view along Z)
    # R_x rotates around X axis (view along X)
    # R_y rotates around Y axis (view along Y)
    # Note: Exact rotation matrices depend on coordinate system convention (LH/RH)
    R_z = rotation_matrix(axis='z', angle=0) 
    R_x = rotation_matrix(axis='x', angle=0) # Or 90 degrees depending on initial alignment
    R_y = rotation_matrix(axis='y', angle=0) # Or 90 degrees depending on initial alignment
    
    # Standard orthographic projection matrix M_clip
    # Maps world coordinates to clip space [-1, 1]
    M_clip = orthographic_projection_matrix(bounding_box)
    
    # Voxel-space matrix M_voxel
    # Maps world coordinates directly to voxel indices [0, N-1]
    M_voxel = voxel_mapping_matrix(bounding_box, grid_resolution)
    
    # Pass 1: View along Z axis
    # Multiply clip-space projection matrix by rotation matrix for Z view
    M_view_z = M_clip * R_z
    render_pass(mesh, M_view_z, M_voxel, voxel_grid, axis='z')
    
    # Pass 2: View along X axis
    M_view_x = M_clip * R_x
    render_pass(mesh, M_view_x, M_voxel, voxel_grid, axis='x')
    
    # Pass 3: View along Y axis
    M_view_y = M_clip * R_y
    render_pass(mesh, M_view_y, M_voxel, voxel_grid, axis='y')
    
    return voxel_grid

def render_pass(mesh, M_view, M_voxel, voxel_grid, axis):
    """
    Render the mesh from a specific view direction and populate the voxel grid.
    """
    for triangle in mesh.triangles:
        # Transform triangle vertices to clip space
        v1_clip = M_view * triangle.v1
        v2_clip = M_view * triangle.v2
        v3_clip = M_view * triangle.v3
        
        # Rasterize triangle to screen/voxel space
        # Determine which voxels are covered by this triangle
        covered_voxels = rasterize_triangle(v1_clip, v2_clip, v3_clip, M_voxel)
        
        for voxel in covered_voxels:
            # Atomic write or simple overwrite
            voxel_grid[voxel.i, voxel.j, voxel.k] = 1 # Occupied
```

**Parameters:**
*   **Resolution:** Higher resolution increases accuracy but also memory and computation cost. Adjusting resolution according to hardware capabilities is a common optimization [S1, unofficial].
*   **View Directions:** Strictly aligned with the principal axes (X, Y, Z) [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Performance:** Requires 3 render calls per object, which becomes expensive as models become more complex [S1, unofficial].
*   **Dynamic Objects:** If optimizing by separating dynamic and static objects, the voxel grid for dynamic objects must be cleared every frame, requiring separate voxel grids for dynamic and static objects [S1, unofficial].
*   **Degeneracy:** Triangles parallel to a view direction may project to a line or point. The 3-pass approach mitigates this by ensuring at least one view has a non-zero projection area, but implementation should handle degenerate triangles gracefully (e.g., skipping or expanding) [S2, Parent Principles, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(3 \cdot N_{tris} \cdot R)$, where $N_{tris}$ is the number of triangles and $R$ is the rasterization cost per triangle. The factor of 3 is constant but significant.
*   **Memory Complexity:** $O(N_x \cdot N_y \cdot N_z)$ for the voxel grid. If dynamic/static separation is used, memory usage doubles [S1, unofficial].
*   **Strengths:** Eliminates cracks and missing data inherent in single-view projection; simple to implement; robust against occlusion [S1, S2, unofficial].
*   **Limitations:** Computationally expensive due to 3 render passes per object; not suitable for real-time applications with very high triangle counts without optimization (e.g., LOD interleaving, resolution scaling) [S1, unofficial].
*   **When to Use:** When geometric completeness is critical and performance budget allows for 3x rasterization cost.
*   **When Not to Use:** When real-time performance is paramount and some geometric inaccuracy (cracks) is acceptable, or when the scene is extremely complex and optimization strategies (like dynamic/static separation) are not feasible.

## Background
3-Pass Voxelization was introduced as a solution to the problem of cracks and missing data in voxel grids generated by single-direction rasterization [S1, unofficial]. It is described in the context of real-time ray-traced voxel global illumination, where accurate voxel representation is crucial for lighting calculations [S1, unofficial]. The method is also implemented in open-source projects like the 3D-Mesh-Voxelizer, demonstrating its practical applicability in real-time graphics pipelines [S2, unofficial].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] 3D-Mesh-Voxelizer - GitHub, https://github.com/Talkative-Banana/3D-Mesh-Voxelizer, fetched 2026-10-26 (unofficial)
