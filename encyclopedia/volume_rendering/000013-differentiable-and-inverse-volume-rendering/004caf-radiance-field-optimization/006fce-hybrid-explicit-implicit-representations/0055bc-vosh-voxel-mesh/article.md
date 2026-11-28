# Vosh (VOxel-meSH)

Also called: Voxel-Mesh Hybrid Representation

## Overview
Vosh is a hybrid neural rendering representation that combines a voxel grid and a triangle mesh to achieve real-time view synthesis on mobile devices [S1, unofficial]. It addresses the trade-off between voxel rendering quality and mesh rasterization speed by converting portions of the volumetric density field into a surface mesh based on rendering quality differences [S1, unofficial]. Unlike VMesh, which relies on Signed Distance Functions (SDF) and manual hyperparameter tuning, Vosh uses only the implicit volumetric density field, making it suitable for unbounded scenes and complex backgrounds [S1, unofficial].

## How it works
Vosh constructs a hybrid representation through a three-stage training process followed by WebGL-based inference [S1, unofficial].

### Inputs and Outputs
*   **Inputs:** A set of multi-view images of the scene [S1, unofficial].
*   **Outputs:** A hybrid asset containing sparse voxels and textured meshes, optimized for real-time rendering [S1, unofficial].

### Data Structures
1.  **Voxel Grid:** A 3D grid storing neural radiance field values (density $\sigma$ and color features $c$) for volumetric regions [S1, unofficial].
2.  **Mesh:** A triangle mesh representing surfaces extracted from the density field, storing vertex positions and texture coordinates [S1, unofficial].
3.  **Hybrid Ratio:** A scalar parameter controlling the balance between voxel and mesh components [S1, unofficial].

### Training Procedure
The training consists of three stages [S1, unofficial]:
1.  **Grid Training:** Initialize a voxel grid model and train it via volume rendering using the multi-view images [S1, unofficial].
2.  **Meshing Density Field:** Calculate the difference in rendering quality between the current voxel grid volume and the potential surfaces it contains [S1, unofficial]. Regions where the rendering quality of the volume is similar to the underlying surface are converted into a mesh, replacing the volumetric density values at those positions [S1, unofficial].
3.  **Optimization:** Perform joint optimization of the remaining voxels and the mesh using hybrid rendering [S1, unofficial]. This stage includes voxel pruning to remove redundant voxels and refine the final hybrid representation [S1, unofficial].

### Inference Phase
The trained assets (sparse voxels and textured meshes) are exported and rendered using WebGL [S1, unofficial]. The mesh component handles simple geometry and textures for fast rendering, while the voxel component handles intricate regions for high-quality rendering [S1, unofficial].

### Formulas
Standard volume rendering integration along a ray $r$ computes the color $C(r)$ as [G]:
$$C(r) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) c_i, \quad T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$$
where $\sigma_i$ is the density, $c_i$ is the color, and $\delta_i$ is the step size between samples [G]. The hybrid rendering blends the mesh rasterization $C_{mesh}(r)$ and voxel volume rendering $C_{voxel}(r)$ using the hybrid ratio $\alpha \in [0, 1]$ [G]:
$$C_{hybrid}(r) = \alpha C_{mesh}(r) + (1 - \alpha) C_{voxel}(r)$$

### Parameters
*   **Hybrid Ratio:** A flexible parameter allowing users to adjust the balance between voxel and mesh components, controlling the trade-off between rendering quality and speed [S1, unofficial].

### Edge Cases and Pitfalls
*   **Complex Backgrounds:** Unlike SDF-based methods (e.g., VMesh), Vosh's density-field-only approach avoids low-quality reconstruction in cluttered backgrounds [S1, unofficial].
*   **Mobile Compatibility:** Vosh avoids CUDA-specific APIs required by 3D Gaussian Splatting, enabling real-time rendering on mobile devices [S1, unofficial].
*   **Precision:** Tri-linear interpolation assumes continuous variation between voxels; sharp discontinuities may require higher resolution or specialized handling [G].
*   **Boundaries:** Points outside the defined storage domain must be handled via clamping or zero-padding [G].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_vosh(multi_view_images, num_epochs, hybrid_ratio):
    # Stage 1: Grid training for initial voxels
    voxel_grid = VoxelGrid(dim=64) # 3D array storing density σ and color features c
    for epoch in range(num_epochs):
        for image, camera_pose in multi_view_images:
            # Standard differentiable volume rendering along rays [G]
            rendered_image = volume_render(voxel_grid, camera_pose)
            loss = mean_squared_error(rendered_image, image)
            voxel_grid.apply_gradients(loss)

    # Stage 2: Meshing density field
    # Determine regions of meshing based on rendering quality difference
    # between the voxel grid volume and potential surfaces it contains [S1, unofficial]
    quality_diff = compute_rendering_quality_difference(voxel_grid, multi_view_images)
    mesh_regions = threshold_regions(quality_diff, threshold=0.1) # [G] Standard thresholding
    # Convert density field in mesh_regions to a textured mesh
    mesh = extract_mesh_from_density(voxel_grid, mesh_regions) # [G] Marching cubes on density field

    # Stage 3: Optimization
    # Joint optimization of voxels and mesh via hybrid rendering and voxel pruning
    for epoch in range(num_epochs):
        for image, camera_pose in multi_view_images:
            # Hybrid rendering: rasterize mesh + volume render remaining voxels
            mesh_color = rasterize_mesh(mesh, camera_pose)
            voxel_color = volume_render(voxel_grid, camera_pose)
            hybrid_image = blend(mesh_color, voxel_color, hybrid_ratio)
            loss = mean_squared_error(hybrid_image, image)
            voxel_grid.apply_gradients(loss)
            mesh.apply_gradients(loss)
            voxel_grid.prune(low_density_threshold=0.01) # Remove redundant voxels [S1, unofficial]
    return voxel_grid, mesh

def render_vosh(voxel_grid, mesh, camera_pose, hybrid_ratio):
    # Inference Phase: Real-time hybrid rendering via WebGL [S1, unofficial]
    mesh_color = rasterize_mesh(mesh, camera_pose)
    voxel_color = volume_render(voxel_grid, camera_pose)
    # Combine based on hybrid ratio and visibility
    final_image = blend(mesh_color, voxel_color, hybrid_ratio)
    return final_image

# Standard operations defined for self-containment
def volume_render(grid, pose): # Standard differentiable volume rendering integration [G]
    pass
def rasterize_mesh(mesh, pose): # Standard triangle rasterization with texture lookup [G]
    pass
def blend(c1, c2, ratio): return ratio * c1 + (1 - ratio) * c2 # [G]
def compute_rendering_quality_difference(grid, images): # [G]
    pass
def threshold_regions(diff, threshold): return diff < threshold # [G]
def extract_mesh_from_density(grid, regions): # [G]
    pass
def prune(grid, threshold): # [G]
    pass
```

## Operating characteristics
*   **Performance:** Achieves real-time view synthesis on mobile devices [S1, unofficial].
*   **Memory:** Reduces space occupation compared to SDF-based methods [S1, unofficial].
*   **Flexibility:** The hybrid ratio allows users to control the balance between rendering quality and speed [S1, unofficial].
*   **Strengths:** Excels in fast rendering for simple geometry/texture via the mesh component and high-quality rendering for intricate regions via the voxel component [S1, unofficial]. Suitable for unbounded outdoor scenes and complex backgrounds [S1, unofficial].
*   **Limitations:** Not given in the sources.
*   **Complexity:** Time and memory complexity are not given in the sources.

## Background
Vosh was proposed to address the limitations of using voxels or meshes individually for neural radiance field-based view synthesis [S1, unofficial]. It was developed as a hybrid representation that optimizes a voxel grid based on neural rendering and strategically meshes a portion of the volumetric density field to surface [S1, unofficial]. The method is designed to enable real-time view synthesis on mobile devices, leveraging WebGL for inference [S1, unofficial].

## Sources
[S1] Voxel-Mesh Hybrid Representation for Real-Time View Synthesis, https://doi.org/10.48550/arxiv.2403.06505, fetched 2026-10-26 (unofficial)
