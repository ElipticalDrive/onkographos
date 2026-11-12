# Plenoxels Training Configuration (PeRFception variants)
Also called: PeRFception-CO3D training, PeRFception-ScanNet training

## Overview
This configuration defines the specific hyperparameters, initialization strategies, and training schedules for optimizing Plenoxels on the CO3D and ScanNet datasets, as introduced in the PeRFception framework. It addresses the instability of standard Plenoxels training on object-centric (CO3D) and indoor (ScanNet) data by adjusting learning rates, loss weights, and voxel pruning thresholds. It differs from the baseline [Plenoxels Training](5979-plenoxels-training.md) by employing a 10× higher sparsity loss lambda, disabling background rendering for ScanNet, and using depth-map-based initialization for ScanNet to mitigate floaters.

## How it works
### Inputs and Outputs
**Inputs:**
*   Training images resized to $640 \times 480$ resolution [S1].
*   Ground truth camera parameters (intrinsics and extrinsics) [S1].
*   Ground truth depth maps (ScanNet only) [S1].
*   Scene type: CO3D (object-centric) or ScanNet (indoor) [S1].

**Outputs:**
*   Optimized voxel grid containing density ($\sigma$) and spherical harmonic (SH) coefficients [S1].

### Data Structures
The method uses the standard Plenoxels voxel grid structure [S1].
*   **Voxel Grid:** A 3D grid of resolution $N^3$.
    *   **ScanNet:** Initialized at $256^3$ [S1].
    *   **CO3D:** Resolution not explicitly specified in the provided notes, but uses the same optimization pipeline [S1].
*   **Voxel Attributes:**
    *   **Density ($\sigma$):** Scalar value per voxel [S1].
    *   **Spherical Harmonic Coefficients:** 9-dimensional vector per RGB channel (27 dimensions total per voxel) [S1].

### Initialization
**CO3D:**
*   Standard Plenoxels initialization is used [S1].

**ScanNet:**
1.  **Depth Unprojection:** Unproject depth maps into a point cloud using ground truth camera parameters [S1].
2.  **Outlier Filtering:**
    *   Discretize the point cloud into coarse voxels with a size of 5 cm [S1].
    *   Perform connected component analysis to identify disconnected voxels [S1].
    *   Remove disconnected voxels, which are likely outliers due to noisy depth observations [S1].
3.  **Grid Initialization:** Use the refined point cloud to initialize the $256^3$ voxel grid [S1].
4.  **Background:** Disable background rendering; all points are treated as foreground [S1].

### Training Schedule and Steps
1.  **Initial Steps (0–1,000):** Skip foreground rendering to stabilize training [S1].
2.  **Optimization:** Gradient-based optimization for 51,200 iterations (ScanNet) or standard duration (CO3D) [S1].
3.  **Voxel Pruning:**
    *   **ScanNet:** Occurs at iteration 25,600 [S1].
    *   **CO3D:** Occurs during the upsampling step [S1].
    *   **Criterion:** Prune voxels where density $\sigma < 1.28$ [S1].

### Loss Functions and Parameters
The total loss combines rendering loss, total variation (TV) regularization, beta loss, and sparsity loss.

**Learning Rates:**
*   Density ($\sigma$): $\eta_\sigma = 30$ [S1].
*   Spherical Harmonic Coefficients ($c$): $\eta_c = 10^{-2}$ [S1].

**Regularization Weights ($\lambda$):**
*   Foreground Density TV: $\lambda_{\sigma, fg} = 5.0 \times 10^{-5}$ [S1].
*   Foreground SH TV: $\lambda_{c, fg} = 5.0 \times 10^{-3}$ [S1].
*   Background Color TV: $\lambda_{color, bg} = 1.0 \times 10^{-3}$ [S1].
*   Background Density TV: $\lambda_{\sigma, bg} = 1.0 \times 10^{-3}$ [S1].
*   Beta Loss: $\lambda_{\beta} = 10^{-5}$ [S1].
*   Sparsity Loss: $\lambda_{sparsity} = 10^{-10}$ [S1]. Note: This is 10 times larger than the default Plenoxels configuration [S1].

**Other Parameters:**
*   Background Brightness: $0.5$ [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_perception_plenoxels(scene_type, images, depths, camera_params):
    # Resize images
    images = resize(images, 640, 480)
    
    # Initialize voxel grid
    if scene_type == "ScanNet":
        # Exclude scenes with severe motion blur
        if detect_motion_blur(images, threshold=10):
            return None # Scene defective
            
        # Initialize with unprojected depth
        point_cloud = unproject_depths(depths, camera_params)
        point_cloud = filter_outliers(point_cloud, voxel_size=0.05) # 5cm
        voxel_grid = init_grid_from_point_cloud(point_cloud, resolution=256)
        background_rendering = False
    else:
        # CO3D: Standard initialization
        voxel_grid = init_grid_standard()
        background_rendering = True
        
    # Hyperparameters
    lr_density = 30.0
    lr_sh = 1e-2
    lambda_tv_sigma_fg = 5.0e-5
    lambda_tv_sh_fg = 5.0e-3
    lambda_tv_color_bg = 1.0e-3
    lambda_tv_sigma_bg = 1.0e-3
    lambda_beta = 1.0e-5
    lambda_sparsity = 1.0e-10
    bg_brightness = 0.5
    prune_threshold = 1.28
    
    # Optimization loop
    for step in range(1, 51201): # ScanNet specific iteration count
        if step <= 1000:
            # Skip foreground rendering for stability
            continue
            
        # Sample rays and render
        rays = sample_rays(camera_params)
        rendered_color, density, sh_coeffs = render(voxel_grid, rays, background_rendering, bg_brightness)
        
        # Compute losses
        loss_render = compute_rendering_loss(rendered_color, ground_truth)
        
        # Total Variation Losses
        loss_tv_sigma_fg = compute_tv(density, region="foreground")
        loss_tv_sh_fg = compute_tv(sh_coeffs, region="foreground")
        if background_rendering:
            loss_tv_color_bg = compute_tv(rendered_color, region="background")
            loss_tv_sigma_bg = compute_tv(density, region="background")
        else:
            loss_tv_color_bg = 0.0
            loss_tv_sigma_bg = 0.0
            
        loss_beta = compute_beta_loss(density)
        loss_sparsity = compute_sparsity_loss(density)
        
        total_loss = (loss_render +
                      lambda_tv_sigma_fg * loss_tv_sigma_fg +
                      lambda_tv_sh_fg * loss_tv_sh_fg +
                      lambda_tv_color_bg * loss_tv_color_bg +
                      lambda_tv_sigma_bg * loss_tv_sigma_bg +
                      lambda_beta * loss_beta +
                      lambda_sparsity * loss_sparsity)
        
        # Update parameters
        update(voxel_grid.density, lr=lr_density, grad=gradient(total_loss, voxel_grid.density))
        update(voxel_grid.sh_coeffs, lr=lr_sh, grad=gradient(total_loss, voxel_grid.sh_coeffs))
        
        # Voxel Pruning
        if scene_type == "ScanNet" and step == 25600:
            prune_voxels(voxel_grid, threshold=prune_threshold)
        elif scene_type == "CO3D" and is_upsampling_step(step):
            prune_voxels(voxel_grid, threshold=prune_threshold)
            
    return voxel_grid

def detect_motion_blur(images, threshold):
    # OpenCV blur detection algorithm
    # Returns True if blur score > threshold
    pass

def filter_outliers(point_cloud, voxel_size):
    # Discretize into coarse voxels
    coarse_voxels = discretize(point_cloud, voxel_size)
    # Connected component analysis
    components = connected_components(coarse_voxels)
    # Remove disconnected voxels (outliers)
    refined_cloud = remove_disconnected(components, point_cloud)
    return refined_cloud
```

## Subtopics

### [PeRFception-ScanNet Data Generation](8193-perfception-scannet-data-generation.md)
Defines the data preparation pipeline for ScanNet, including depth unprojection and outlier filtering. Chosen when training on indoor ScanNet data to handle noisy depth and sparse geometry.

### [Plenoxels Training with Progressive Growing](1493-plenoxels-training-with-progressive-growing.md)
Uses progressive resolution growth. Chosen when higher resolution is needed without the specific ScanNet/CO3D hyperparameter tuning.

### [Joint Finetuning for Vector-Quantized Radiance Fields](1871-joint-finetuning-for-vector-quantized-radiance-fields.md)
Optimizes vector-quantized fields. Chosen for compressed representation training.

### [Plenoxels Training](5979-plenoxels-training.md)
The baseline configuration. Chosen for standard datasets like Tanks and Temples without the specific PeRFception adjustments.

### [Voxel optimization (F2Plenoxels)](7384-voxel-optimization-f2Plenoxels.md)
Focuses on voxel-level optimization techniques. Chosen for fine-grained control over voxel updates.

### [RGB-D Plenoxel Mapping](7820-rgb-d-plenoxel-mapping.md)
Incorporates RGB-D data directly. Chosen when depth data is available and used for initialization/mapping.

### [Plenoxel](8054-plenoxel.md)
The core method. Chosen as the base architecture.

### [Clonoxels](8364-clonoxels.md)
Uses cloning operations. Chosen for dynamic scene reconstruction.

### [Plenoxels Transmittance Offset](10129-plenoxels-transmittance-offset.md)
Adjusts transmittance calculation. Chosen to improve rendering accuracy in specific scenarios.

## Operating characteristics
*   **Time Complexity:** Training takes up to 30 minutes per scene on a single RTX 3090 GPU [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The configuration is tuned for CO3D (18,618 scenes) and ScanNet (1,513 scenes, 16 excluded for blur) [S1].
*   **Strengths:** Stabilizes training on object-centric and indoor data; reduces floaters in ScanNet via depth initialization; improves sparsity control via higher lambda [S1].
*   **Limitations:** Requires ground truth depth for ScanNet initialization; excludes scenes with motion blur [S1].
*   **When to use:** When training Plenoxels on CO3D or ScanNet datasets using the PeRFception framework.
*   **When not to use:** When training on datasets without depth maps (ScanNet-specific initialization fails) or when computational budget is extremely limited (30 mins/scene is higher than some real-time methods).

## Background
The PeRFception framework was introduced in the paper "PeRFception: Perception using Radiance Fields" [S1]. It adapts the Plenoxels method [14] for perception tasks on CO3D and ScanNet datasets. The configuration addresses specific challenges such as motion blur, sparse observation, and floaters in empty space [S1].

## Sources
[S1] PeRFception: Perception using Radiance Fields, https://doi.org/10.48550/arxiv.2208.11537, fetched 2026-10-26
