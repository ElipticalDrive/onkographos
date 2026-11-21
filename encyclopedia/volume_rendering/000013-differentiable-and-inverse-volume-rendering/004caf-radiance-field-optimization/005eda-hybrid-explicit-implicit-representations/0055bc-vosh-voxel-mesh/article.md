# Vosh (VOxel-meSH)

Also called: Voxel-Mesh Hybrid Representation

## Overview
Vosh is a hybrid neural rendering representation that combines a voxel grid and a triangle mesh to achieve real-time view synthesis on mobile devices [S1, unofficial]. It solves the trade-off between the high-quality rendering of voxels and the fast rasterization of meshes by converting portions of the volumetric density field into a surface mesh based on rendering quality differences [S1, unofficial]. Unlike VMesh, which relies on Signed Distance Functions (SDF) and manual hyperparameter tuning, Vosh uses only the implicit volumetric density field, making it suitable for unbounded scenes and complex backgrounds [S1, unofficial].

## How it works
Vosh constructs a hybrid representation through a three-stage training process followed by WebGL-based inference [S1, unofficial].

### Inputs and Outputs
*   **Inputs:** A set of multi-view images of the scene.
*   **Outputs:** A hybrid asset containing sparse voxels and textured meshes, optimized for real-time rendering [S1, unofficial].

### Data Structures
1.  **Voxel Grid:** A 3D grid storing neural radiance field values (density and color features) for volumetric regions [S1, unofficial].
2.  **Mesh:** A triangle mesh representing surfaces extracted from the density field, storing vertex positions and texture coordinates [S1, unofficial].
3.  **Hybrid Ratio:** A flexible parameter controlling the balance between voxel and mesh components [S1, unofficial].

### Training Procedure
The training consists of three stages [S1, unofficial]:

1.  **Grid Training:** Initialize a voxel grid model and train it via volume rendering using the multi-view images [S1, unofficial].
2.  **Meshing Density Field:** Calculate the difference in rendering quality between the current voxel grid volume and the potential surfaces it contains. Regions where the rendering quality of the volume is similar to the underlying surface are converted into a mesh, replacing the volumetric density values at those positions [S1, unofficial].
3.  **Optimization:** Perform joint optimization of the remaining voxels and the mesh using hybrid rendering. This stage includes voxel pruning to remove redundant voxels and refine the final hybrid representation [S1, unofficial].

### Inference Phase
The trained assets (sparse voxels and textured meshes) are exported and rendered using WebGL [S1, unofficial]. The mesh component handles simple geometry and textures for fast rendering, while the voxel component handles intricate regions for high-quality rendering [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vosh Training and Inference Logic
# Based on [S1, unofficial]

def train_vosh(multi_view_images):
    # Stage 1: Grid training for initial voxels
    voxel_grid = initialize_voxel_grid()
    voxel_grid = train_volume_rendering(voxel_grid, multi_view_images)
    
    # Stage 2: Meshing density field
    # Determine regions for meshing based on rendering quality difference
    # between the volume and its underlying surface
    mesh_regions = identify_meshing_regions(voxel_grid, multi_view_images)
    
    # Convert density field in mesh_regions to a mesh
    mesh = extract_mesh_from_density(voxel_grid, mesh_regions)
    
    # Stage 3: Optimization
    # Optimize voxels and mesh via hybrid rendering and voxel pruning
    voxel_grid, mesh = optimize_hybrid_representation(
        voxel_grid, 
        mesh, 
        multi_view_images,
        hybrid_ratio=0.5 # Placeholder for flexible parameter
    )
    
    return voxel_grid, mesh

def render_vosh(voxel_grid, mesh, camera_pose):
    # Inference Phase: Real-time hybrid rendering via WebGL
    # Rasterize mesh for simple geometry
    mesh_color = rasterize_mesh(mesh, camera_pose)
    
    # Volume render voxels for intricate regions
    voxel_color = volume_render_voxels(voxel_grid, camera_pose)
    
    # Combine based on hybrid ratio and visibility
    final_image = combine_hybrid_rendering(mesh_color, voxel_color)
    return final_image
```

### Parameters
*   **Hybrid Ratio:** A flexible parameter allowing users to adjust the balance between voxel and mesh components, controlling the trade-off between rendering quality and speed [S1, unofficial].

### Edge Cases and Pitfalls
*   **Complex Backgrounds:** Unlike SDF-based methods (e.g., VMesh), Vosh's density-field-only approach avoids low-quality reconstruction in cluttered backgrounds [S1, unofficial].
*   **Mobile Compatibility:** Vosh avoids CUDA-specific APIs required by 3D Gaussian Splatting, enabling real-time rendering on mobile devices [S1, unofficial].

## Operating characteristics
*   **Performance:** Achieves real-time view synthesis on mobile devices [S1, unofficial].
*   **Memory:** Reduces space occupation compared to SDF-based methods [S1, unofficial].
*   **Flexibility:** The hybrid ratio allows users to control the balance between rendering quality and speed [S1, unofficial].
*   **Strengths:** Excels in fast rendering for simple geometry/texture via the mesh component and high-quality rendering for intricate regions via the voxel component [S1, unofficial]. Suitable for unbounded outdoor scenes and complex backgrounds [S1, unofficial].
*   **Limitations:** Not given in the sources.

## Background
Vosh was proposed to address the limitations of using voxels or meshes individually for neural radiance field-based view synthesis [S1, unofficial]. It was developed as a hybrid representation that optimizes a voxel grid based on neural rendering and strategically meshes a portion of the volumetric density field to surface [S1, unofficial]. The method is designed to enable real-time view synthesis on mobile devices, leveraging WebGL for inference [S1, unofficial].

## Sources
[S1] Voxel-Mesh Hybrid Representation for Real-Time View Synthesis, https://doi.org/10.48550/arxiv.2403.06505, fetched 2026-10-26 (unofficial)
