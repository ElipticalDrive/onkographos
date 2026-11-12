# DirectVoxGO

Also called: Direct Voxel Grid Optimization, DVGO

## Overview
DirectVoxGO is a neural radiance field implementation that uses explicit 3D voxel grids to store density and color features, enabling super-fast convergence for radiance field reconstruction [S1, unofficial]. It distinguishes itself from implicit NeRF alternatives by optimizing the voxel grid values directly rather than a neural network's weights, which significantly reduces training time [S1, unofficial]. It supports both standard dense grids and FourierGrid variants, featuring occupancy caching and near-camera voxel masking to handle unbounded scenes and prevent artifacts [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Source images, camera poses (from COLMAP), bounding box defined by `xyz_min` and `xyz_max`, and configuration parameters (grid resolution, MLP dimensions, etc.).
*   **Outputs:** Optimized density grid, color feature grid (`k0`), and optionally an MLP for color prediction.

**Data Structures:**
1.  **Density Grid:** A 3D tensor storing density values. Can be `DenseGrid` or `FourierGrid`.
2.  **Color Feature Grid (`k0`):** A 3D tensor storing color features. Dimensions depend on `rgbnet_dim`.
3.  **Mask Grid (`mask_cache`):** A 3D boolean tensor indicating occupied voxels to skip computation.
4.  **MLP (`rgbnet`):** Optional multi-layer perceptron for color prediction, taking color features and view direction as input.

**Numbered Steps:**

1.  **Initialize Density Grid:**
    *   If `use_fourier_grid` is false: Create grid using `grid.create_grid` with `density_type`, `channels=1`, `world_size`, `xyz_min`, `xyz_max`, and `density_config`.
    *   If `use_fourier_grid` is true: Create grid using `FourierGrid_grid.create_grid` with additional `use_nerf_pos=True` and `fourier_freq_num` parameters.

2.  **Initialize Color Feature Grid (`k0`):**
    *   If `rgbnet_dim <= 0`: Set `k0_dim=3`. Create grid with `channels=3`. This is the coarse stage, using no MLP.
    *   If `rgbnet_dim > 0`: Set `k0_dim` based on `rgbnet_dim` and flags. Create grid with `channels=k0_dim`. Initialize MLP (`rgbnet`) for fine stage.

3.  **Initialize MLP Input Dimension (`dim0`):**
    *   Start with `3 + 3 * viewbase_pe * 2` (position + view direction PE).
    *   If `rgbnet_direct` is true: Add `k0_dim`.
    *   If `rgbnet_direct` is false and not `full_implicit`: Add `k0_dim - 3`.
    *   Initialize MLP weights with ReLU activations and zero bias on the final layer.

4.  **Initialize Mask Cache:**
    *   If `mask_cache_path` is provided: Load `MaskGrid` and compute mask for grid points.
    *   Otherwise: Create mask of all `True` values.
    *   Store as `grid.MaskGrid` with `xyz_min` and `xyz_max`.

5.  **Mask Out Near-Camera Voxels:**
    *   **Non-FourierGrid:** Compute grid `xyz` coordinates. For each camera origin (split into batches of 100 for memory), compute nearest distance from grid points to camera origin. Set density grid values to `-100` where `nearest_dist <= near_clip`.
    *   **FourierGrid:** Compute normalized coordinates and apply `nerf_pos` embedding. For each position embedding dimension, compute nearest distance from grid points to camera origin in the embedded space. Set density grid values to `-100` where `nearest_dist <= near_clip`.

6.  **Scale Volume Grid:**
    *   Update `world_size` and `voxel_size`.
    *   Scale density and `k0` grids.
    *   If `total_voxels <= 256^3`: Compute alpha from density, apply 3D max pooling (kernel=3, padding=1, stride=1), and update `mask_cache` by combining with `alpha > fast_color_thres`.

7.  **Update Occupancy Cache:**
    *   Compute grid `xyz` for `mask_cache` resolution.
    *   Query density and activate to get alpha.
    *   Apply 3D max pooling (kernel=3, padding=1, stride=1).
    *   Update `mask_cache` mask by ANDing with `alpha > fast_color_thres`.

**Formulas:**
*   Activation shift: $\alpha_{init} = \log(\frac{1}{\alpha_{init}} - 1)$ [S1, unofficial]
*   Base voxel size: $voxel\_size_{base} = \sqrt[3]{\frac{Volume}{num_{voxels_{base}}}}$ [S1, unofficial]
*   Current voxel size: $voxel\_size = \sqrt[3]{\frac{Volume}{num_{voxels}}}$ [S1, unofficial]
*   World size: $world\_size = \lfloor \frac{xyz_{max} - xyz_{min}}{voxel\_size} \rfloor$ [S1, unofficial]
*   Voxel size ratio: $voxel\_size_{ratio} = \frac{voxel\_size}{voxel\_size_{base}}$ [S1, unofficial]

**Parameters:**
*   `xyz_min`: Lower bound of the 3D world coordinate system, stored as a `torch.Tensor` buffer [S1, unofficial].
*   `xyz_max`: Upper bound of the 3D world coordinate system, stored as a `torch.Tensor` buffer [S1, unofficial].
*   `num_voxels`: Total number of voxels in the primary grid resolution [S1, unofficial].
*   `num_voxels_base`: Reference voxel count used to calculate base voxel size for resolution scaling [S1, unofficial].
*   `alpha_init`: Initial alpha value used to compute the density bias shift for the activation function [S1, unofficial].
*   `mask_cache_thres`: Threshold value for the mask cache to determine occupancy [S1, unofficial].
*   `fast_color_thres`: Threshold for determining if a voxel is opaque enough to skip color network evaluation [S1, unofficial].
*   `density_type`: String specifying the type of density grid (e.g., 'DenseGrid') [S1, unofficial].
*   `k0_type`: String specifying the type of color feature grid [S1, unofficial].
*   `rgbnet_dim`: Dimension of the color feature grid. If `<= 0`, uses a coarse stage with 3-channel grid and no MLP; otherwise uses fine stage with feature grid and MLP [S1, unofficial].
*   `rgbnet_direct`: Boolean flag for direct color encoding in the MLP input [S1, unofficial].
*   `rgbnet_full_implicit`: Boolean flag for full implicit color encoding [S1, unofficial].
*   `rgbnet_depth`: Depth of the color MLP (number of layers) [S1, unofficial].
*   `rgbnet_width`: Width (hidden dimension) of the color MLP [S1, unofficial].
*   `viewbase_pe`: Number of positional encoding frequencies for view direction [S1, unofficial].

**Pitfalls:**
*   **Memory usage during near-camera masking:** Camera origins are split into batches of 100 to prevent excessive memory consumption during distance calculations [S1, unofficial].
*   **FourierGrid near-camera masking:** Uses position embedding dimensions for distance calculation instead of raw coordinates, requiring iteration over `pos_embed_output_dim` [S1, unofficial].
*   **Max pooling for occupancy:** 3D max pooling with kernel 3, padding 1, stride 1 is applied to alpha values to smooth occupancy and prevent thin structures from being missed [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def initialize_density_grid(use_fourier_grid, density_type, world_size, xyz_min, xyz_max, density_config, fourier_freq_num=None):
    """
    Initialize the density grid based on configuration.
    """
    if not use_fourier_grid:
        # Assuming grid.create_grid is a helper function defined elsewhere or standard
        # In a real implementation, this would call the specific grid class
        density_grid = create_grid(density_type, channels=1, world_size=world_size, 
                                   xyz_min=xyz_min, xyz_max=xyz_max, config=density_config)
    else:
        density_grid = FourierGrid.create_grid(use_nerf_pos=True, fourier_freq_num=fourier_freq_num,
                                               channels=1, world_size=world_size, 
                                               xyz_min=xyz_min, xyz_max=xyz_max)
    return density_grid

def initialize_color_grid(rgbnet_dim, k0_type, world_size, xyz_min, xyz_max):
    """
    Initialize the color feature grid (k0) and optionally the MLP.
    """
    if rgbnet_dim <= 0:
        k0_dim = 3
        k0_grid = create_grid(k0_type, channels=3, world_size=world_size, 
                              xyz_min=xyz_min, xyz_max=xyz_max)
        mlp = None
    else:
        k0_dim = rgbnet_dim # Simplified assumption based on notes
        k0_grid = create_grid(k0_type, channels=k0_dim, world_size=world_size, 
                              xyz_min=xyz_min, xyz_max=xyz_max)
        mlp = initialize_mlp(k0_dim)
    return k0_grid, mlp, k0_dim

def initialize_mlp(k0_dim, rgbnet_direct=False, rgbnet_full_implicit=False, 
                   rgbnet_depth=3, rgbnet_width=128, viewbase_pe=2):
    """
    Initialize the MLP for color prediction.
    """
    # Input dimension calculation
    dim0 = 3 + 3 * viewbase_pe * 2
    if rgbnet_direct:
        dim0 += k0_dim
    elif not rgbnet_full_implicit:
        dim0 += k0_dim - 3
    
    # Simple MLP structure
    layers = []
    for i in range(rgbnet_depth):
        in_dim = dim0 if i == 0 else rgbnet_width
        out_dim = rgbnet_width
        layers.append(torch.nn.Linear(in_dim, out_dim))
        if i < rgbnet_depth - 1:
            layers.append(torch.nn.ReLU())
        else:
            # Zero bias on final layer as per notes
            layers[-1].bias.data.zero_()
            
    return torch.nn.Sequential(*layers)

def initialize_mask_cache(mask_cache_path, world_size, xyz_min, xyz_max):
    """
    Initialize the occupancy mask cache.
    """
    if mask_cache_path is not None:
        # Load existing mask
        mask_cache = load_mask_grid(mask_cache_path, xyz_min, xyz_max)
    else:
        # Create all True mask
        mask_cache = torch.ones(world_size, dtype=torch.bool, device='cpu') # Simplified
        mask_cache = MaskGrid(mask_cache, xyz_min, xyz_max)
    return mask_cache

def mask_near_camera_voxels(density_grid, xyz_min, xyz_max, cameras, near_clip, use_fourier_grid=False, fourier_freq_num=None):
    """
    Mask out voxels too close to the camera.
    """
    # Get grid coordinates
    grid_coords = get_grid_coordinates(density_grid.shape, xyz_min, xyz_max)
    
    # Batch cameras to save memory
    batch_size = 100
    for i in range(0, len(cameras), batch_size):
        batch_cameras = cameras[i:i+batch_size]
        for cam in batch_cameras:
            cam_origin = cam.origin
            
            if not use_fourier_grid:
                # Compute distance in raw coordinates
                distances = compute_distance_to_origin(grid_coords, cam_origin)
            else:
                # Compute distance in Fourier embedded space
                pos_embed = apply_nerf_pos(grid_coords, fourier_freq_num)
                distances = compute_distance_to_origin_embedded(pos_embed, cam_origin, fourier_freq_num)
            
            # Mask voxels
            mask = distances <= near_clip
            density_grid[mask] = -100.0

def scale_volume_grid(density_grid, k0_grid, xyz_min, xyz_max, num_voxels, num_voxels_base, fast_color_thres):
    """
    Scale the volume grid and update mask cache.
    """
    volume = (xyz_max - xyz_min).prod()
    
    # Calculate voxel sizes
    voxel_size_base = (volume / num_voxels_base) ** (1/3)
    voxel_size = (volume / num_voxels) ** (1/3)
    
    # Update world size
    world_size = torch.floor((xyz_max - xyz_min) / voxel_size).int()
    
    # Scale grids (interpolation or resizing)
    # Note: Exact scaling method depends on grid implementation
    density_grid = resize_grid(density_grid, world_size)
    k0_grid = resize_grid(k0_grid, world_size)
    
    # Update mask cache if small enough
    if num_voxels <= 256**3:
        alpha = activate_density(density_grid)
        # 3D max pooling
        alpha_pooled = F.max_pool3d(alpha.unsqueeze(0).unsqueeze(0), kernel_size=3, stride=1, padding=1).squeeze()
        mask_update = alpha_pooled > fast_color_thres
        # Combine with existing mask (AND operation)
        # mask_cache.update(mask_update)
        return density_grid, k0_grid, mask_update
    return density_grid, k0_grid, None

def update_occupancy_cache(density_grid, mask_cache, fast_color_thres):
    """
    Update the occupancy cache based on density.
    """
    # Query density and activate
    alpha = activate_density(density_grid)
    
    # 3D max pooling
    alpha_pooled = F.max_pool3d(alpha.unsqueeze(0).unsqueeze(0), kernel_size=3, stride=1, padding=1).squeeze()
    
    # Update mask
    mask_update = alpha_pooled > fast_color_thres
    mask_cache.mask = mask_cache.mask & mask_update
    return mask_cache

def activate_density(density_grid):
    """
    Convert density to alpha using the shift formula.
    """
    # alpha_init is a parameter
    shift = torch.log(1.0 / 0.01 - 1) # Example alpha_init = 0.01
    density_shifted = density_grid + shift
    alpha = 1.0 - torch.exp(-torch.relu(density_shifted))
    return alpha
```

## Subtopics

### [SPC-NeRF](12634-spc-nerf.md)
SPC-NeRF is a Rate-Distortion (RD) optimized predictive coding framework designed to compress Explicit Voxel Grid (EVG) represented Neural Radiance Fields (NeRFs) by removing spatial redundancy. It distinguishes itself from sibling methods like DirectVoxGO by introducing a two-stage finetuning process and a reference graph that jointly optimizes compression ratio and rendering distortion during training, achieving 32% bit savings compared to the state-of-the-art VQRF method.

## Operating characteristics
*   **Time Complexity:** Training is significantly faster than implicit NeRFs due to direct optimization of voxel values, avoiding the need for many MLP evaluations per ray [S1, unofficial].
*   **Memory Complexity:** Memory usage is proportional to the number of voxels ($N^3$) and the feature dimensions. Near-camera masking requires batching camera origins to manage memory spikes [S1, unofficial].
*   **Measured Performance:** The method achieves "super-fast convergence" compared to standard NeRF optimization [S1, unofficial]. Specific metrics (PSNR, SSIM, training time) are not provided in the sources.
*   **Scaling:** Supports scaling of the volume grid by adjusting `num_voxels` and `world_size` [S1, unofficial].
*   **Strengths:** Fast training, explicit representation allows for easy manipulation and compression, handles unbounded scenes with near-camera masking [S1, unofficial].
*   **Limitations:** Memory usage can be high for large scenes or high resolutions. FourierGrid variant adds complexity to near-camera masking [S1, unofficial].
*   **When to use:** When fast training convergence is prioritized over the potentially higher quality of implicit NeRFs, or when explicit voxel manipulation is needed.
*   **When not to use:** For very large scenes where memory constraints are tight and sparse representations (like [Sparse Voxel Grid Optimization](2568-sparse-voxel-grid-optimization.md)) are more appropriate.

## Background
DirectVoxGO was introduced by Cheng Sun, Min Sun, and Hwann-Tzong Chen in the paper "Direct voxel grid optimization: Super-fast convergence for radiance fields reconstruction" presented at the IEEE/CVF Conference on Computer Vision and Pattern Recognition (CVPR) in 2022 [S1, unofficial]. It builds on the idea of optimizing voxel grids directly, offering a faster alternative to implicit neural representations. The code is available on GitHub [S1, unofficial].

## Sources
[S1] Direct voxel grid optimization: Super-fast convergence for radiance fields reconstruction, https://github.com/sjtuytc/UnboundedNeRFPytorch, fetched 2026-10-26 (unofficial)
