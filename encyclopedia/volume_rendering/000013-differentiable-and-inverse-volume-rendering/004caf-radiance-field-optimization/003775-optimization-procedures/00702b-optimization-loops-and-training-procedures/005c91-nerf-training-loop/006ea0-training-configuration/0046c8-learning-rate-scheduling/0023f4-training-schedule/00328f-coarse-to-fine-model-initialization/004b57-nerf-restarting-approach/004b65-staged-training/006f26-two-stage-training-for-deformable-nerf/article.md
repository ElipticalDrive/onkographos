# Two-stage Training for Deformable NeRF

Also called: Deformable NeRF Two-stage Training

## Overview
A training protocol for deformable neural radiance fields that reduces GPU memory consumption and accelerates convergence by decoupling the initialization of the tetrahedral mesh from the high-resolution refinement. It differs from single-stage training by first constructing a sparse, occupancy-aware coarse mesh and then refining it, whereas single-stage methods initialize with a dense, uniform grid regardless of scene occupancy.

## How it works
The method optimizes a deformable neural radiance field represented by a recursively subdivided tetrahedral mesh. The training is split into two distinct phases to manage memory and computational load.

### Inputs and Outputs
*   **Inputs:**
    *   Camera poses and images from a dataset (e.g., Synthetic NeRF, Tanks and Temples).
    *   A unit cube bounding box for the scene.
    *   Hyperparameters: Grid spacing for initialization ($\delta = 0.02$), total subdivision levels ($L=6$), optimizer settings (Adam, $\beta_1=0.9, \beta_2=0.999, \epsilon=10^{-15}$), initial learning rate ($\eta = 1 \times 10^{-2}$), and cosine annealing schedule.
*   **Outputs:**
    *   A refined tetrahedral mesh with associated neural field parameters (density, color, deformation fields) optimized for novel view synthesis.

### Data Structures
The core data structure is a tetrahedral mesh.
*   **Vertices:** Defined in 3D space.
*   **Tetrahedra:** Defined by 4 vertex indices.
*   **Subdivision Hierarchy:** A tree structure where each tetrahedron can be subdivided into smaller tetrahedra up to level $L$. The coefficient matrix $C$ relates barycentric coordinates across levels (derived in supplementary material of [S1]).
*   **Occupancy Mask:** A boolean array indicating which tetrahedra are retained.

### Steps

**Stage 1: Coarse Mesh Initialization**
1.  **Generate Initial Tetrahedralization:** Apply the QuarTet algorithm [17] to the unit cube with a grid spacing parameter of 0.02. This creates a uniform tetrahedral mesh filling the cube.
    *   *Result:* A mesh with 17,933 vertices and 92,234 tetrahedra [S1].
2.  **Compute Occupancy:** Render the scene or use a pre-computed occupancy mesh to determine which tetrahedra intersect with the object geometry.
3.  **Prune Mesh:** Retain only tetrahedra marked as occupied. Discard empty tetrahedra to minimize memory usage.
    *   *Goal:* Obtain a coarse tetrahedral mesh that roughly envelops the object [S1].

**Stage 2: Refined Training**
1.  **Initialize Subdivision:** Start with the coarse mesh from Stage 1. Set the total number of subdivision levels to $L=6$ for all scenes [S1].
2.  **Optimize:** Perform gradient-based optimization (Adam) on the neural field parameters associated with the retained tetrahedra.
    *   The subdivision levels allow for higher resolution in areas requiring detail.
    *   The optimization is focused on refining the structure established in Stage 1.
3.  **Schedule:** Use a cosine annealing learning rate schedule starting at $1 \times 10^{-2}$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def two_stage_training(dataset, scene_bbox_size=1.0, grid_spacing=0.02, 
                       total_levels=6, lr=1e-2, beta1=0.9, beta2=0.999, 
                       epsilon=1e-15):
    """
    Two-stage training for Deformable NeRF.
    
    Args:
        dataset: Collection of images and camera poses.
        scene_bbox_size: Size of the unit cube (default 1.0).
        grid_spacing: QuarTet grid spacing (default 0.02).
        total_levels: Total subdivision levels (default 6).
        lr: Initial learning rate (default 1e-2).
        beta1, beta2, epsilon: Adam optimizer parameters.
        
    Returns:
        optimized_model: Neural radiance field model with refined mesh.
    """
    
    # --- Stage 1: Coarse Initialization ---
    
    # 1. Generate initial uniform tetrahedral mesh using QuarTet
    # QuarTet converts surface geometry (or bounding box) into tetrahedra.
    # Input: Cube of size scene_bbox_size, grid spacing grid_spacing.
    initial_mesh = quarTet_init(cube_size=scene_bbox_size, spacing=grid_spacing)
    
    # Note: Initial mesh has ~17,933 vertices and ~92,234 tetrahedra for spacing 0.02 [S1]
    
    # 2. Determine occupancy
    # Use an occupancy mesh or rendering to identify occupied tetrahedra.
    # Similar to Instant-NGP's occupancy grid for voxel retention [S1].
    occupancy_mask = compute_occupancy(initial_mesh, dataset)
    
    # 3. Prune mesh
    # Retain only tetrahedra where occupancy_mask is True.
    coarse_mesh = prune_mesh(initial_mesh, occupancy_mask)
    
    # Initialize model parameters with coarse_mesh
    model = initialize_model(coarse_mesh, levels=total_levels)
    
    # --- Stage 2: Refined Optimization ---
    
    optimizer = Adam(model.parameters(), lr=lr, betas=(beta1, beta2), eps=epsilon)
    scheduler = CosineAnnealingLR(optimizer, T_max=total_epochs) # T_max depends on dataset
    
    for epoch in range(total_epochs):
        # Sample rays from dataset
        rays = sample_rays(dataset)
        
        # Render rays using current model (which uses subdivided tetrahedra)
        # The subdivision hierarchy allows detailed representation [S1]
        rendered_colors, depths = render_rays(model, rays)
        
        # Compute loss against ground truth images
        loss = compute_loss(rendered_colors, rays.gt_colors)
        
        # Backpropagate and update
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        scheduler.step()
        
    return model
```

### Parameters and Typical Values
*   **Grid Spacing:** 0.02 [S1].
*   **Total Subdivision Levels:** 6 for all scenes [S1].
*   **Learning Rate:** $1 \times 10^{-2}$ [S1].
*   **Optimizer:** Adam with $\beta_1=0.9, \beta_2=0.999, \epsilon=10^{-15}$ [S1].

### Edge Cases and Pitfalls
*   **Occupancy Accuracy:** If the occupancy mesh is inaccurate, Stage 1 may retain empty tetrahedra (wasting memory) or discard occupied ones (causing holes). The method relies on the quality of the occupancy estimation [S1].
*   **Memory Scaling:** While two-stage training reduces memory, Stage 2 still requires significant resources for high subdivision levels. However, it is more efficient than single-stage [S1].
*   **Degenerate Input:** The method assumes the scene can be bounded by a unit cube. Scenes outside this bounds require scaling or padding [S1].

## Operating characteristics
*   **Memory Usage:** Significantly lower than single-stage training. On the Synthetic NeRF dataset, Stage 1 uses 5.34 GB and Stage 2 uses 2.70 GB, compared to 9.79 GB for single-stage training [S1].
*   **Performance:** Improves rendering quality. On the Tanks and Temples dataset, two-stage training achieves PSNR 29.09, SSIM 0.903, and LPIPS 0.124, compared to single-stage PSNR 28.01, SSIM 0.887, and LPIPS 0.155 [S1].
*   **Speed:** Enhances training speed by reducing the number of tetrahedra processed in the initial phase and focusing computation on relevant areas [S1].
*   **Strengths:** Memory efficiency, faster convergence, better handling of complex geometries via occupancy-aware pruning.
*   **Limitations:** Requires an initial occupancy estimation step. Performance is sensitive to the grid spacing parameter and number of subdivision levels.

## Background
*   **Origin:** Proposed in "Deformable NeRF using Recursively Subdivided Tetrahedra" [S1].
*   **Context:** Builds on Tetra-NeRF [16] and Instant-NGP [27]. It addresses the high memory cost of dense tetrahedral grids in deformable NeRFs.
*   **History:** The method leverages the QuarTet algorithm [17] for mesh generation and integrates it with a multi-resolution subdivision hierarchy.

## Sources
[S1] Deformable NeRF using Recursively Subdivided Tetrahedra, https://doi.org/10.1145/3664647.3681019, fetched 2026-10-26
