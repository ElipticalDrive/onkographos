# Plenoxels Training Configuration (PeRFception variants)

Also called: PeRFception-CO3D training, PeRFception-ScanNet training

## Overview
This configuration defines the specific hyperparameters, initialization strategies, and training schedules required to optimize Plenoxels for the CO3D and ScanNet datasets, as implemented in the PeRFception framework [S1]. It addresses dataset-specific challenges such as sparse observation, motion blur, and background ambiguity by modifying learning rates, loss weights, and voxel initialization methods [S1]. It differs from standard Plenoxels configurations by employing distinct initialization pipelines for ScanNet (depth-based) versus CO3D (default), disabling background rendering for indoor scenes, and applying a sparsity loss ten times stronger than the default [S1].

## How it works
The method optimizes voxel opacity ($\sigma$) and spherical harmonic (SH) coefficients using gradient-based optimization [S1]. The implementation relies on the official Plenoxels PyTorch implementation with custom CUDA kernels [S1].

### Inputs and Outputs
**Inputs:**
*   Training images resized to $640 \times 480$ resolution [S1].
*   Ground truth camera parameters (intrinsics and extrinsics) [S1].
*   Ground truth depth maps (required for ScanNet initialization) [S1].
*   Scene mask indicating foreground/background (ScanNet treats all points as foreground) [S1].

**Outputs:**
*   Optimized voxel grid containing density values ($\sigma$) and 9-dimensional SH coefficients per RGB channel [S1].

### Data Structures
The voxel grid is a 3D structure. For ScanNet, the initial resolution is $256^3$ [S1]. The grid stores:
1.  **Density ($\sigma$):** Scalar value per voxel [S1].
2.  **Spherical Harmonics (SH):** 9 coefficients per RGB channel (27 values total per voxel) [S1].

### Initialization
Initialization depends on the dataset variant:

**PeRFception-CO3D:**
Uses the default Plenoxels initialization [S1].

**PeRFception-ScanNet:**
1.  **Depth Unprojection:** Unproject depth maps into a point cloud using ground truth camera parameters [S1].
2.  **Outlier Filtering:**
    *   Discretize the point cloud into coarse voxels with a size of 5 cm [S1].
    *   Perform connected component analysis to identify disconnected voxels [S1].
    *   Remove disconnected voxels, which are likely outliers due to noisy depth observations [S1].
3.  **Grid Initialization:** Initialize the $256^3$ voxel grid using the refined point cloud [S1].

### Training Schedule and Steps
1.  **Blur Detection (ScanNet only):** Automatically exclude scenes with severe motion blur using OpenCV's blur detection algorithm with a threshold of 10 [S1]. Scenes with too few frames after filtering are considered defective [S1].
2.  **Warm-up Phase:** Skip foreground rendering for the first 1,000 training steps to stabilize training [S1].
3.  **Optimization Loop:** Run for 51,200 iterations [S1].
4.  **Voxel Pruning:** At iteration 25,600, prune voxels whose density values fall below 1.28 [S1].
5.  **Background Rendering:**
    *   **ScanNet:** Disable background rendering; all points are treated as foreground [S1].

### Parameters and Loss Functions
The optimization minimizes a composite loss function [S1]. The specific weights and learning rates are:

**Learning Rates:**
*   Density values ($\sigma$): 30 [S1].
*   Spherical Harmonic coefficients: $10^{-2}$ [S1].

**Loss Weights ($\lambda$):**
*   Total Variation (TV) loss for foreground densities: $5.0 \times 10^{-5}$ [S1].
*   Total Variation (TV) loss for foreground SH coefficients: $5.0 \times 10^{-3}$ [S1].
*   Total Variation (TV) loss for background colors: $1.0 \times 10^{-3}$ [S1].
*   Total Variation (TV) loss for background densities: $1.0 \times 10^{-3}$ [S1].
*   Beta loss: $10^{-5}$ [S1].
*   Sparsity loss: $10^{-10}$ (10 times larger than the default configuration) [S1].

**Other Parameters:**
*   Background brightness: 0.5 [S1].

### Pseudocode
The following pseudocode outlines the training loop and initialization for the PeRFception variants [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_plenoxels_perfception(dataset_type, scenes, device='cuda'):
    for scene in scenes:
        # 1. Pre-processing and Initialization
        if dataset_type == 'ScanNet':
            if compute_blur_score(scene.images) > 10:
                continue
            point_cloud = unproject_depth(scene.depth_maps, scene.cameras)
            point_cloud = filter_outliers(point_cloud, voxel_size=0.05)
            voxel_grid = initialize_grid_from_pointcloud(point_cloud, resolution=256)
            background_rendering = False
        else:
            voxel_grid = initialize_grid_default()
            background_rendering = True

        lr_density = 30.0
        lr_sh = 1e-2
        lambda_tv_density_fg = 5.0e-5
        lambda_tv_sh_fg = 5.0e-3
        lambda_tv_color_bg = 1.0e-3
        lambda_tv_density_bg = 1.0e-3
        lambda_beta = 1.0e-5
        lambda_sparsity = 1.0e-10
        bg_brightness = 0.5
        prune_threshold = 1.28
        prune_iteration = 25600
        total_iterations = 51200
        warmup_steps = 1000

        for step in range(total_iterations):
            rays = sample_rays(scene)
            if step < warmup_steps:
                rendered_color = render_background_only(rays, voxel_grid, bg_brightness)
            else:
                rendered_color = render_full(rays, voxel_grid, bg_brightness, background_rendering)

            target_color = rays.target_color
            loss_recon = mean_squared_error(rendered_color, target_color)

            loss_tv_fg_density = total_variation(voxel_grid.density, mask=foreground_mask)
            loss_tv_fg_sh = total_variation(voxel_grid.sh, mask=foreground_mask)

            loss_tv_bg_color = total_variation(voxel_grid.bg_color) if background_rendering else 0.0
            loss_tv_bg_density = total_variation(voxel_grid.bg_density) if background_rendering else 0.0

            loss_tv = (lambda_tv_density_fg * loss_tv_fg_density +
                       lambda_tv_sh_fg * loss_tv_fg_sh +
                       lambda_tv_color_bg * loss_tv_bg_color +
                       lambda_tv_density_bg * loss_tv_bg_density)

            loss_beta = beta_loss(voxel_grid)
            loss_sparsity = sparsity_loss(voxel_grid)

            total_loss = loss_recon + loss_tv + lambda_beta * loss_beta + lambda_sparsity * loss_sparsity

            total_loss.backward()

            update_optimizer(voxel_grid.density, lr=lr_density)
            update_optimizer(voxel_grid.sh, lr=lr_sh)

            if step == prune_iteration:
                prune_voxels(voxel_grid, threshold=prune_threshold)

def compute_blur_score(images):
    return variance_of_laplacian(images[0])

def unproject_depth(depth_maps, cameras):
    points = []
    for depth, cam in zip(depth_maps, cameras):
        points.extend(depth_to_pointcloud(depth, cam))
    return points

def filter_outliers(point_cloud, voxel_size=0.05):
    grid = discretize(point_cloud, voxel_size)
    components = connected_components(grid)
    return remove_disconnected(components, point_cloud)

def initialize_grid_from_pointcloud(point_cloud, resolution):
    grid = empty_grid(resolution)
    for p in point_cloud:
        grid.set_density_at(p, initial_value=1.0)
    return grid

def initialize_grid_default():
    return empty_grid(resolution=256)

def sample_rays(scene):
    return generate_rays(scene)

def render_background_only(rays, grid, bg_brightness):
    return rays.alpha * bg_brightness

def render_full(rays, grid, bg_brightness, background_rendering):
    if background_rendering:
        return rays.alpha * grid.color + (1 - rays.alpha) * bg_brightness
    return rays.alpha * grid.color

def mean_squared_error(pred, target):
    return sum((p - t) ** 2 for p, t in zip(pred, target)) / len(pred)

def total_variation(tensor, mask=None):
    if mask is not None:
        tensor = tensor * mask
    return sum(abs(tensor[i+1] - tensor[i]) for i in range(len(tensor)-1)) / len(tensor)

def beta_loss(grid):
    return sum(d ** 2 for d in grid.density) / len(grid.density)

def sparsity_loss(grid):
    return sum(d for d in grid.density) / len(grid.density)

def update_optimizer(tensor, lr):
    tensor.data -= lr * tensor.grad
```

## Subtopics

### [PeRFception-ScanNet Data Generation](8193-perfception-scannet-data-generation.md)
Defines the data preparation pipeline for ScanNet, using depth-based initialization and CPU-cached ray batches to handle sparse observations and motion blur artifacts.

### [PERF with Cauchy Loss and Hierarchical Planes](15328-perf-with-cauchy-loss-and-hierarchical-planes.md)
Extends the PERF algorithm with difference planes and Cauchy loss to mitigate shape-radiance ambiguity, separating view-independent density from view-dependent appearance.

## Operating characteristics
*   **Time Complexity:** Training takes up to 30 minutes per scene on a single RTX 3090 GPU [S1].
*   **Memory Complexity:** Not given in the sources [S1].
*   **Performance:** The configuration is tuned for the CO3D (18,618 scenes) and ScanNet (1,513 scenes, 16 excluded) datasets [S1].
*   **Strengths:** Stabilizes training via warm-up and specific learning rates; handles ScanNet's noisy depth data through robust outlier filtering; improves sparsity control via increased sparsity loss weight [S1].
*   **Limitations:** Resolution does not directly correlate with average rendering quality on ScanNet (tested at 128, 256, 512) [S1]. Fails to render unobserved parts reliably if camera poses are too distant from training views [S1].
*   **When to use:** When training Plenoxels on CO3D or ScanNet datasets where standard configurations fail to converge or produce artifacts due to sparse data or motion blur [S1].

## Background
The PeRFception variants were introduced in the PeRFception paper [S1]. The framework builds on the Plenoxels method [S1], which uses PyTorch with custom CUDA kernels [S1]. The configurations adapt to dataset-specific properties, such as noisy depth observations in ScanNet and standard object-centric captures in CO3D [S1].

## Sources
[S1] PeRFception: Perception using Radiance Fields, https://doi.org/10.48550/arxiv.2208.11537, fetched 2026-10-26
