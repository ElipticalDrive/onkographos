# VGOS

Also called: Voxel Grid Optimization for View Synthesis from Sparse Inputs

## Overview
VGOS is a method for fast radiance field reconstruction from sparse inputs (3-10 views) that addresses overfitting, holes, and floaters in voxel grids by using an incremental voxel training strategy and voxel smoothing regularization [S1]. It sets itself apart from pre-training methods like SRF and MVSNeRF by training from scratch without pre-trained models or extra inputs, and from depth-dependent methods like DSNeRF by relying solely on RGB images [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3-10 sparse RGB images with known camera poses [S1].
*   **Outputs:** A reconstructed radiance field represented by optimized voxel grids [S1].

**Data Structures:**
The method builds upon the Direct Voxel Grid Optimization structure, utilizing a dense voxel grid for density and a feature voxel grid for color [S1, Parent]. The specific layout of these grids is inherited from the parent method [Parent].

**Algorithm Steps:**
1.  **Initialization:** Initialize the voxel grids for density and features [Parent].
2.  **Incremental Voxel Training:**
    *   Define a set of "peripheral voxels" located near the cameras' near planes [S1].
    *   During the early stage of training, freeze the optimization (gradients) of these peripheral voxels [S1].
    *   Gradually thaw (enable optimization for) the peripheral voxels as training progresses [S1]. This strategy prevents overfitting to training views [S1].
3.  **Voxel Smoothing Regularization:**
    *   Render depth maps from unobserved viewpoints [S1].
    *   Apply a **depth smooth loss** to regularize the dense voxel grids [S1]. This eliminates holes and floaters [S1].
    *   Apply a **color-aware voxel smoothness loss** to penalize sharpness inside the voxel grids [S1]. This avoids degenerate solutions [S1].
4.  **Optimization:** Update voxel parameters using the combined losses [S1].

**Parameters:**
*   **Input Views:** 3-10 sparse RGB images [S1].
*   **Training Time:** 3-5 minutes per scene [S1].

**Edge Cases and Pitfalls:**
*   **Overfitting:** Sparse inputs are prone to overfitting to training views, leading to artifacts [S1]. The incremental training strategy mitigates this [S1].
*   **Holes and Floaters:** Voxel grids may contain holes and floaters in sparse scenarios [S1]. The voxel smoothing method addresses this [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for VGOS optimization loop
# Inherits voxel grid structures from Direct Voxel Grid Optimization

def train_vgos(images, cameras, num_steps):
    # Initialize voxel grids (density and features)
    density_grid = initialize_density_grid()
    feature_grid = initialize_feature_grid()
    
    # Identify peripheral voxels (near camera near planes)
    peripheral_voxels = identify_peripheral_voxels(cameras)
    
    for step in range(num_steps):
        # 1. Incremental Voxel Training Strategy
        # Freeze peripheral voxels in early stage, gradually thaw
        if step < early_stage_threshold:
            freeze_gradients(density_grid, peripheral_voxels)
            freeze_gradients(feature_grid, peripheral_voxels)
        else:
            # Gradually thaw based on progress
            thaw_progress = (step - early_stage_threshold) / (num_steps - early_stage_threshold)
            thaw_gradients(density_grid, peripheral_voxels, thaw_progress)
            thaw_gradients(feature_grid, peripheral_voxels, thaw_progress)
            
        # Sample rays and render
        rays = sample_rays(cameras)
        rendered_colors, rendered_depths = render_rays(
            density_grid, feature_grid, rays
        )
        
        # 2. Compute Losses
        # Color loss (standard)
        color_loss = compute_color_loss(rendered_colors, images)
        
        # Depth smooth loss (from unobserved viewpoints)
        # Note: Sources mention regularizing depth maps from unobserved viewpoints
        unobserved_depths = render_depth_from_unobserved_views(
            density_grid, cameras
        )
        depth_smooth_loss = compute_depth_smoothness(unobserved_depths)
        
        # Color-aware voxel smoothness loss
        voxel_smooth_loss = compute_color_aware_voxel_smoothness(
            density_grid, feature_grid
        )
        
        # Total loss
        total_loss = color_loss + depth_smooth_loss + voxel_smooth_loss
        
        # Update voxel grids
        update_gradients(density_grid, feature_grid, total_loss)
        
    return density_grid, feature_grid
```

## Operating characteristics
*   **Time Complexity:** Training time is 3-5 minutes per scene [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** On the LLFF dataset in 3-view settings, VGOS achieves higher PSNR and lower training time than DietNeRF, DVGOv2, SRF, MVSNeRF, and RegNeRF, measured on a single NVIDIA RTX 3090 GPU [S1].
*   **Strengths:** Fast convergence (one to two orders of magnitude speedup compared to SOTA approaches with on-par quality) [S1]. No need for pre-trained models or extra inputs [S1].
*   **Limitations:** Performance is specific to sparse input scenarios (3-10 views) [S1].
*   **When to use:** When fast reconstruction from sparse RGB inputs is required, and pre-training or depth inputs are not available [S1].
*   **When not to use:** When dense inputs are available, or when pre-trained priors are acceptable for potentially higher quality [S1].

## Background
VGOS was proposed in 2023 by authors publishing in the Proceedings of the Thirty-Second International Joint Conference on Artificial Intelligence (IJCAI-23) [S1]. It addresses the limitation of Neural Radiance Fields (NeRF) which require tens to hundreds of dense inputs and hours to days of training [S1]. VGOS builds on Direct Voxel Grid Optimization [Parent] to enable fast radiance field reconstruction from sparse inputs [S1].

## Sources
[S1] VGOS: Voxel Grid Optimization for View Synthesis from Sparse Inputs, https://doi.org/10.24963/ijcai.2023/157, fetched 2026-10-26
