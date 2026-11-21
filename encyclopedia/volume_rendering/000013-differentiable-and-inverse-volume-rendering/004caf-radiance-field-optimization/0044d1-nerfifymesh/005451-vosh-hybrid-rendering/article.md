# Vosh Hybrid Rendering

Also called: Vosh, Voxel-Mesh Hybrid Representation

## Overview
Vosh is a hybrid representation method that combines a textured mesh with a sparse voxel grid to achieve real-time view synthesis while preserving comprehensive scene information [S1, unofficial]. It solves the trade-off between the geometric precision of meshes and the volumetric completeness of neural radiance fields by using the mesh for surface details and the voxels for thin structures and background volume [S1, unofficial]. Unlike Adaptive Shells, which constrain volume rendering to regions near the surface, Vosh allows pure volume rendering where necessary, making it suitable for outdoor scenes [S1, unofficial]. It differs from VMesh by reconstructing a volumetric density field, enabling the mesh to represent thin structures like fishing nets that are difficult to capture with Signed Distance Fields (SDFs) [S1, unofficial].

## How it works
Vosh operates in three stages: initial voxel training, mesh refinement, and final hybrid optimization. The method ensures consistency between GPU training and web inference by employing both rasterization and ray marching passes [S1, unofficial].

### Inputs and Outputs
*   **Inputs**: Textured mesh models, camera poses, and multi-view images.
*   **Outputs**: A hybrid asset consisting of a textured mesh and a sparse voxel grid, along with associated metadata for real-time rendering.

### Data Structures
1.  **Mesh**: Standard triangle mesh with appearance textures storing diffuse color and view-dependent features.
2.  **Sparse Voxel Grid**: Based on the multi-resolution hierarchical occupancy grid from SNeRG [S1, unofficial].
    *   **Occupancy Grid**: Binary mask images at multiple resolution levels created via max-pooling to mark occupied voxels.
    *   **Voxel Attributes**: Stored in a sparse structure to minimize memory usage.
3.  **Mesh-Occupancy Grid**: A grid used to filter spatial overlap between mesh and voxels. Resolution $r_{mesh}$ is a hyperparameter.

### Parameters
*   $r_{mesh}$: Resolution of the mesh-occupancy grid. Set to 128 for Vosh-Light [S1, unofficial].
*   $\lambda_{voxel}$: Voxel pruning loss weight. Set to 0.001 for Vosh-Base and 0.01 for Vosh-Light [S1, unofficial].
*   $premove$: Parameter for mesh refinement stage. Set to 0.9 for Vosh-Base and 1.0 for Vosh-Light [S1, unofficial].
*   Marching Cubes Resolution: 1024 [S1, unofficial].
*   Optimizer: 25,000 steps, initial learning rate $1e-2$, warmup for 100 steps, decaying to $1e-3$ [S1, unofficial].

### Algorithm Steps

#### Stage 1: Initial Voxel Training
1.  Train initial voxels using classic MSE loss to measure pixel-level color differences [S1, unofficial].
2.  Use the same optimizer settings as MERF [S1, unofficial].

#### Stage 2: Mesh Refinement
1.  Perform meshing of the density field for 20,000 steps [S1, unofficial].
2.  Set $premove$ to 0.9 (Vosh-Base) or 1.0 (Vosh-Light) [S1, unofficial].
3.  Clean the mesh using operations similar to NeRF2Mesh [S1, unofficial].
4.  Apply L2 regularization terms with rendering weights for background colors to encourage voxels to fill holes in the mesh [S1, unofficial].
5.  Repeat mesh cleaning operations if messy floating faces are produced [S1, unofficial].

#### Stage 3: Hybrid Optimization and Rendering
This stage combines rasterization and ray marching.

1.  **Calculate Mesh-Occupancy Grid**:
    *   Compute the occupancy grid of the meshes under training poses [S1, unofficial].
    *   Filter out voxels located in the mesh-occupied grid areas during rendering and baking to prevent spatial overlap [S1, unofficial].
    *   Adjust the ratio of sparse voxels by setting $r_{mesh}$ to suppress voxel density [S1, unofficial].

2.  **Rasterization Pass**:
    *   Uniformly shade the meshes.
    *   The fragment shader samples appearance textures to incorporate diffuse color and view-dependent features [S1, unofficial].
    *   Output: Diffuse color, view-dependent features, and a depth map [S1, unofficial].

3.  **Ray Marching Pass**:
    *   Use the multi-resolution hierarchical occupancy grid (from SNeRG) to create binary mask images via max-pooling [S1, unofficial].
    *   Perform spatial skips during ray marching using these masks [S1, unofficial].
    *   Reduce sampling distance using the depth map obtained from the rasterization pass [S1, unofficial].

4.  **Hybrid Combination**:
    *   For each spatial coordinate after ray marching, compare the transformed depth with the rasterization depth map [S1, unofficial].
    *   Determine if the ray has intersected the surface position [S1, unofficial].
    *   If the ray reaches the mesh surface:
        *   Stop marching [S1, unofficial].
        *   Weightily combine the accumulated ray marching result with the appearance result from the rasterization pass [S1, unofficial].
    *   If the ray does not hit the surface, continue marching or accumulate volume contribution as per standard volume rendering [G].

5.  **Loss Function**:
    *   Apply voxel pruning loss weighted by $\lambda_{voxel}$ [S1, unofficial].
    *   For Vosh-Base, disable the binary mesh-occupancy grid [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vosh_render(camera, mesh, voxel_grid, mesh_occupancy_grid, depth_map_raster):
    """
    Renders a frame using Vosh hybrid representation.
    
    Args:
        camera: Camera parameters (position, orientation, intrinsics).
        mesh: Textured mesh object.
        voxel_grid: Sparse voxel grid with multi-resolution occupancy masks.
        mesh_occupancy_grid: Binary grid indicating mesh-occupied voxels.
        depth_map_raster: Depth map from the rasterization pass.
        
    Returns:
        color: Rendered pixel color.
    """
    
    # 1. Rasterization Pass
    # Uniformly shade meshes and sample appearance textures
    raster_output = rasterize_mesh(mesh, camera)
    diffuse_color = raster_output.diffuse_color
    view_dependent_features = raster_output.view_dependent_features
    depth_map_raster = raster_output.depth_map
    
    # 2. Ray Marching Setup
    # Get ray directions for each pixel
    rays = generate_rays(camera)
    
    # Initialize accumulation
    accumulated_color = 0.0
    accumulated_alpha = 0.0
    
    for each pixel (u, v):
        ray = rays[u, v]
        current_depth = 0.0
        ray_marching_accum = 0.0
        ray_marching_alpha = 0.0
        
        # 3. Ray Marching with Spatial Skips
        # Use multi-resolution occupancy grid for spatial skips
        while current_depth < max_distance:
            # Check occupancy grid for spatial skip
            if not voxel_grid.is_occupied(ray, current_depth):
                current_depth += skip_step
                continue
            
            # Sample voxel density and color
            density, color = voxel_grid.sample(ray, current_depth)
            
            # Standard volume rendering accumulation
            alpha = 1.0 - exp(-density * step_size)
            ray_marching_accum += color * alpha * (1.0 - ray_marching_alpha)
            ray_marching_alpha += alpha * (1.0 - ray_marching_alpha)
            
            current_depth += step_size
            
            # 4. Surface Intersection Check
            # Compare transformed depth with rasterization depth map
            raster_depth = depth_map_raster[u, v]
            
            # If ray hits mesh surface (within tolerance)
            if abs(current_depth - raster_depth) < epsilon:
                # Stop marching
                break
        
        # 5. Hybrid Combination
        # Weightily combine ray marching result with rasterization appearance
        # Note: The exact weighting function is not specified in sources [G]
        # Assuming a simple blend based on alpha or depth proximity
        mesh_color = diffuse_color[u, v]
        
        # Combine: If surface hit, use mesh color weighted by ray marching alpha
        # If no surface hit, use ray marching result
        if surface_hit:
            color = lerp(ray_marching_accum, mesh_color, ray_marching_alpha)
        else:
            color = ray_marching_accum
            
        accumulated_color[u, v] = color
        
    return accumulated_color

def vosh_train(mesh, voxel_grid, images, poses):
    """
    Trains the Vosh hybrid representation.
    """
    # Stage 1: Initial Voxel Training
    train_initial_voxels(voxel_grid, images, poses, loss_fn=MSE)
    
    # Stage 2: Mesh Refinement
    refine_mesh(mesh, voxel_grid, premove=0.9, steps=20000)
    clean_mesh(mesh)
    apply_l2_regularization(voxel_grid, mesh, target='background_colors')
    
    # Stage 3: Hybrid Optimization
    for step in range(25000):
        # Calculate mesh-occupancy grid
        mesh_occupancy_grid = calculate_mesh_occupancy(mesh, poses)
        
        # Filter voxels in mesh-occupied areas
        voxel_grid.filter(mesh_occupancy_grid)
        
        # Render and compute loss
        rendered_images = vosh_render_all_views(mesh, voxel_grid, mesh_occupancy_grid, images, poses)
        loss = compute_loss(rendered_images, images)
        
        # Add voxel pruning loss
        if model_type == 'Vosh-Light':
            loss += lambda_voxel * compute_voxel_pruning_loss(voxel_grid)
            
        # Backpropagate
        optimizer.step(loss)
```

## Operating characteristics
*   **Training Time**: On a single NVIDIA RTX 4090, training takes approximately 66 minutes for initial voxels, 35 minutes for mesh refinement, and 58 minutes for Vosh optimization [S1, unofficial].
*   **Memory Usage**: Peak VRAM usage during training is ~11 GB [S1, unofficial].
*   **Storage**:
    *   Vosh-Base: 506 MB VRAM, 488 MB disk [S1, unofficial].
    *   Vosh-Light: 490 MB VRAM, 232 MB disk [S1, unofficial].
    *   These sizes are on Mip-NeRF 360 outdoor scenes [S1, unofficial].
*   **Performance**: Vosh achieves real-time rendering on mobile devices [S1, unofficial].
*   **Strengths**:
    *   Restores comprehensive scene information compared to SDF-based methods [S1, unofficial].
    *   Can represent thin structures (e.g., fishing nets, ropes) that are difficult for SDFs [S1, unofficial].
    *   Suitable for outdoor scenes where pure volume rendering is necessary [S1, unofficial].
*   **Limitations**:
    *   Slightly lower rendering quality in indoor scenes compared to MLP-based methods due to the use of a tiny view-dependent MLP [S1, unofficial].
    *   Disk storage is slightly larger than MERF because it does not use triplane representation [S1, unofficial].
*   **When to Use**: When real-time rendering on mobile devices is required, and the scene contains thin structures or requires pure volume rendering (e.g., outdoor scenes).
*   **When Not to Use**: When maximum indoor rendering quality is prioritized over speed, or when disk space is extremely constrained and MERF's triplane representation is acceptable.

## Background
Vosh (Voxel-Mesh Hybrid Representation) was introduced in the paper "Voxel-Mesh Hybrid Representation for Real-Time View Synthesis" [S1, unofficial]. The method builds on previous works such as SNeRG for sparse voxel storage [S1, unofficial], MERF for optimizer settings [S1, unofficial], and NeRF2Mesh for mesh cleaning [S1, unofficial]. It was developed to address the limitations of pure mesh or pure volume representations, particularly in the context of real-time view synthesis on mobile devices [S1, unofficial].

## Sources
[S1] Voxel-Mesh Hybrid Representation for Real-Time View Synthesis, https://doi.org/10.48550/arxiv.2403.06505, fetched 2026-10-26 (unofficial)
