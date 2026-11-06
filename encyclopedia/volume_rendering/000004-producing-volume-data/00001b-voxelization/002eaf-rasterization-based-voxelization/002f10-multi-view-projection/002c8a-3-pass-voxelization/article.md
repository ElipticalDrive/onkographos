# 3-Pass Voxelization

Also called: 3-Axis Voxelization, 3D Mesh Voxelization via Axis-Aligned Rasterization, Multi-View Voxelization

## Overview
3-Pass Voxelization is a rasterization-based technique that eliminates cracks and missing data in voxel grids by rendering the scene from three distinct viewpoints aligned with the principal axes (X, Y, Z) [S1, S2, unofficial]. Unlike single-view projection methods which suffer from geometric distortion or gaps when triangle normals are parallel to the view direction, this method ensures complete coverage by combining the occupancy data from all three orthogonal passes [S2, unofficial]. It is chosen when topological integrity (6-separation) is critical and the performance cost of multiple draw calls is acceptable [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A mesh defined by vertices and indices; a voxel grid of fixed resolution $R_x \times R_y \times R_z$; a bounding box defining the spatial extent of the scene.
- **Output:** A populated voxel grid where each voxel indicates occupancy (and optionally material/normal data).

**Data Structures:**
- **Voxel Grid:** A 3D array or texture buffer. For atomic writes in parallel implementations, this must support atomic operations [G].
- **Projection Matrices:**
  - $M_{clip}$: The standard clip-space projection matrix (perspective or orthographic).
  - $M_{voxel}$: A matrix that maps world-space coordinates directly to voxel indices. It typically scales by the grid resolution and translates by the bounding box minimum.
  - $R_x, R_y, R_z$: Rotation matrices aligning the camera with the X, Y, and Z axes respectively.

**Steps:**
1.  **Setup Coordinate System:** Place the origin at the center of the voxel grid, ensuring the principal axes are parallel to the grid edges [S2, unofficial].
2.  **Pass 1 (Axis 1):**
    - Construct the view matrix for the first axis (e.g., looking along the Z-axis).
    - Compute the combined projection matrix: $M_{view1} = M_{voxel} \times R_{axis1}$. Note that $M_{voxel}$ already handles the mapping to voxel space, so the rotation is applied to the view direction to generate fragments from the correct perspective [S1, unofficial].
    - Render the scene. Fragments are mapped to voxel indices using $M_{view1}$.
3.  **Pass 2 (Axis 2):**
    - Construct the view matrix for the second axis (e.g., looking along the X-axis).
    - Compute $M_{view2} = M_{voxel} \times R_{axis2}$.
    - Render the scene.
4.  **Pass 3 (Axis 3):**
    - Construct the view matrix for the third axis (e.g., looking along the Y-axis).
    - Compute $M_{view3} = M_{voxel} \times R_{axis3}$.
    - Render the scene.
5.  **Merge:** The voxel grid accumulates occupancy from all three passes. If a voxel is hit in any pass, it is marked as occupied [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxelize_3_pass(mesh, voxel_grid, resolution, bounding_box_min, bounding_box_max):
    # Calculate voxel space matrix: scales to resolution, translates to min
    scale = resolution / (bounding_box_max - bounding_box_min)
    M_voxel = create_scale_translate_matrix(scale, -bounding_box_min)
    
    # Define rotation matrices for X, Y, Z axes
    # These rotate the world so the respective axis points towards the camera
    R_x = rotation_matrix_x(90) 
    R_y = rotation_matrix_y(90)
    R_z = rotation_matrix_z(0) # Or 180 depending on handedness
    
    axes = [R_x, R_y, R_z]
    
    for R_axis in axes:
        # Combine voxel mapping with axis-specific rotation
        # M_voxel handles coordinate-to-index mapping; R_axis handles view direction
        M_final = M_voxel @ R_axis
        
        # Set shader uniforms with M_final
        set_uniform("MVP", M_final)
        
        # Render mesh to voxel grid
        # The fragment shader computes voxel index from clip space or world space
        # and performs an atomic write to the grid
        render(mesh, voxel_grid)
```

**Edge Cases and Pitfalls:**
- **Performance:** Requires 3 render calls per object, which becomes expensive for complex models [S1, unofficial].
- **Dynamic Objects:** If optimizing by separating static and dynamic objects, dynamic objects must be cleared every frame. This requires maintaining separate voxel grids for static and dynamic content [S1, unofficial].
- **Precision:** Ensure the projection matrix does not introduce floating-point errors that cause gaps between adjacent voxels [G].

## Operating characteristics
- **Time Complexity:** $O(N \cdot V)$, where $N$ is the number of triangles and $V$ is the number of voxels (or more accurately, proportional to the number of fragments generated, which is roughly $3 \times$ the screen-space area of the mesh projected onto the voxel grid).
- **Memory Complexity:** $O(R_x R_y R_z)$ for the voxel grid.
- **Strengths:** Guarantees 6-separation (tunnel-free volume) by covering all principal axes; simple to implement; robust against orientation-dependent gaps [S1, S2, unofficial].
- **Limitations:** High computational cost due to multiple passes; not suitable for real-time applications with very high triangle counts without LOD optimizations [S1, unofficial].
- **When to use:** When voxel quality is paramount and performance allows for multiple passes.
- **When not to use:** For highly dynamic scenes with complex geometry where 3 passes per frame are prohibitive.

## Background
3-Pass Voxelization was developed as a solution to the "cracks" problem in single-view rasterization-based voxelization. It is described in the context of real-time ray-traced voxel global illumination [S1, unofficial] and implemented in open-source 3D mesh voxelizers [S2, unofficial]. The method leverages the redundancy of multi-view projection to ensure that no surface is parallel to all view directions, thereby guaranteeing coverage.

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] 3D-Mesh-Voxelizer - GitHub, https://github.com/Talkative-Banana/3D-Mesh-Voxelizer, fetched 2026-10-26 (unofficial)
