# DirectVoxGO

Also called: DirectVoxelGridOptimization

## Overview
DirectVoxGO is a differentiable volume rendering method that optimizes a single, dense 3D voxel grid for density and color features, enabling super-fast convergence for radiance field reconstruction [S1, unofficial]. It distinguishes itself from hierarchical methods like [Hierarchical Grid Optimization](2567-hierarchical-grid-optimization.md) by avoiding multi-resolution data structures, instead relying on efficient occupancy caching and near-camera voxel masking to handle unbounded scenes and accelerate training [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- Source images and their corresponding camera poses (extrinsics and intrinsics).
- Scene bounding box defined by `xyz_min` and `xyz_max`.
- Hyperparameters: `num_voxels`, `alpha_init`, `mask_cache_thres`, `fast_color_thres`, `rgbnet_dim`, `rgbnet_depth`, `rgbnet_width`, `viewbase_pe`, `rgbnet_direct`, `rgbnet_full_implicit`, `use_fourier_grid`, `fourier_freq_num`.

**Outputs:**
- Optimized density grid (scalar field).
- Optimized color feature grid (vector field).
- Optional MLP weights (if `rgbnet_dim > 0`).
- Rendered images for a given camera pose.

### Data Structures
1.  **Density Grid:** A 3D tensor of shape `(world_size, world_size, world_size)` storing density values. Type is determined by `density_type` (e.g., `DenseGrid`).
2.  **Color Feature Grid (k0):** A 3D tensor of shape `(world_size, world_size, world_size, k0_dim)`.
    - If `rgbnet_dim <= 0` (coarse stage), `k0_dim = 3` and no MLP is used.
    - If `rgbnet_dim > 0` (fine stage), `k0_dim` is derived from `rgbnet_dim` and flags.
3.  **Mask Cache:** A 3D boolean tensor of shape `(mask_cache_res, mask_cache_res, mask_cache_res)` indicating occupied voxels. Stored as `grid.MaskGrid` with `xyz_min` and `xyz_max`.
4.  **MLP:** A multi-layer perceptron for color prediction, initialized with ReLU activations and zero bias on the final layer.

### Initialization Steps
1.  **Initialize Density Grid:**
    - If `use_fourier_grid` is false: Create grid using `grid.create_grid` with `density_type`, `channels=1`, `world_size`, `xyz_min`, `xyz_max`, and `density_config`.
    - If `use_fourier_grid` is true: Create grid using `FourierGrid_grid.create_grid` with additional `use_nerf_pos=True` and `fourier_freq_num`.
2.  **Initialize Color Feature Grid (k0):**
    - If `rgbnet_dim <= 0`: Set `k0_dim = 3`. Create grid with `channels=3`. No MLP is initialized.
    - If `rgbnet_dim > 0`: Set `k0_dim` based on `rgbnet_dim` and flags. Create grid with `channels=k0_dim`. Initialize MLP (`rgbnet`) with `rgbnet_depth` layers and `rgbnet_width` width.
3.  **Initialize MLP Input Dimension (dim0):**
    - Start with `dim0 = 3 + 3 * viewbase_pe * 2`.
    - If `rgbnet_direct` is true, add `k0_dim`.
    - If `rgbnet_direct` is false and not `full_implicit`, add `k0_dim - 3`.
4.  **Initialize Mask Cache:**
    - If `mask_cache_path` is provided, load `MaskGrid` and compute mask for grid points.
    - Otherwise, create a mask of all `True` values.
    - Store as `grid.MaskGrid` with `xyz_min` and `xyz_max`.
5.  **Mask Out Near-Camera Voxels:**
    - **Non-FourierGrid:** Compute grid XYZ coordinates. For each camera origin (split into batches of 100 for memory), compute the nearest distance from grid points to the camera origin. Set density grid values to `-100` where `nearest_dist <= near_clip`.
    - **FourierGrid:** Compute normalized coordinates and apply NeRF position embedding. For each position embedding dimension, compute the nearest distance from grid points to the camera origin in the embedded space. Set density grid values to `-100` where `nearest_dist <= near_clip`.
6.  **Scale Volume Grid:**
    - Update `world_size` and `voxel_size`.
    - Scale density and k0 grids.
    - If `total_voxels <= 256^3`: Compute alpha from density, apply 3D max pooling (kernel=3, padding=1, stride=1), and update `mask_cache` by combining with `alpha > fast_color_thres`.
7.  **Update Occupancy Cache:**
    - Compute grid XYZ for `mask_cache` resolution.
    - Query density and activate to get alpha.
    - Apply 3D max pooling (kernel=3, padding=1, stride=1).
    - Update `mask_cache` mask by ANDing with `alpha > fast_color_thres`.

### Rendering Formula
During rendering, rays are sampled through the voxel grid. The density $\sigma$ is activated using:
$$ \sigma = \text{activation}(\text{density} + \text{act\_shift}) $$
where:
$$ \text{act\_shift} = \log\left(\frac{1}{\alpha_{init}} - 1\right) $$
The color $C$ is computed by querying the color feature grid and passing through the MLP (if active) along with view direction positional encoding.

### Parameters
- `xyz_min`: Lower bound of the 3D world coordinate system, stored as a `torch.Tensor` buffer.
- `xyz_max`: Upper bound of the 3D world coordinate system, stored as a `torch.Tensor` buffer.
- `num_voxels`: Total number of voxels in the primary grid resolution.
- `num_voxels_base`: Reference voxel count used to calculate base voxel size for resolution scaling.
- `alpha_init`: Initial alpha value used to compute the density bias shift for the activation function.
- `mask_cache_thres`: Threshold value for the mask cache to determine occupancy.
- `fast_color_thres`: Threshold for determining if a voxel is opaque enough to skip color network evaluation.
- `density_type`: String specifying the type of density grid (e.g., 'DenseGrid').
- `k0_type`: String specifying the type of color feature grid.
- `rgbnet_dim`: Dimension of the color feature grid. If `<= 0`, uses a coarse stage with 3-channel grid and no MLP; otherwise uses fine stage with feature grid and MLP.
- `rgbnet_direct`: Boolean flag for direct color encoding in the MLP input.
- `rgbnet_full_implicit`: Boolean flag for full implicit color encoding.
- `rgbnet_depth`: Depth of the color MLP (number of layers).
- `rgbnet_width`: Width (hidden dimension) of the color MLP.
- `viewbase_pe`: Number of positional encoding frequencies for view direction.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def initialize_direct_voxgo(xyz_min, xyz_max, num_voxels, alpha_init, 
                            rgbnet_dim, rgbnet_direct, rgbnet_full_implicit,
                            viewbase_pe, use_fourier_grid, fourier_freq_num,
                            near_clip, fast_color_thres):
    """
    Initialize the DirectVoxGO model structures.
    """
    # Calculate world size and voxel size
    volume = torch.prod(xyz_max - xyz_min)
    voxel_size = (volume / num_voxels).pow(1/3)
    world_size = torch.floor((xyz_max - xyz_min) / voxel_size).long()
    
    # Initialize Density Grid
    if not use_fourier_grid:
        density_grid = create_dense_grid(world_size, channels=1, xyz_min=xyz_min, xyz_max=xyz_max)
    else:
        density_grid = create_fourier_grid(world_size, channels=1, xyz_min=xyz_min, xyz_max=xyz_max,
                                           use_nerf_pos=True, fourier_freq_num=fourier_freq_num)
    
    # Initialize Color Feature Grid (k0)
    if rgbnet_dim <= 0:
        k0_dim = 3
        k0_grid = create_dense_grid(world_size, channels=3, xyz_min=xyz_min, xyz_max=xyz_max)
        mlp = None
    else:
        k0_dim = rgbnet_dim # Simplified assumption based on notes
        k0_grid = create_dense_grid(world_size, channels=k0_dim, xyz_min=xyz_min, xyz_max=xyz_max)
        mlp = initialize_mlp(k0_dim, viewbase_pe, rgbnet_direct, rgbnet_full_implicit)
        
    # Initialize Mask Cache
    mask_cache = create_mask_grid(world_size, xyz_min, xyz_max)
    
    # Mask out near-camera voxels
    mask_near_camera(density_grid, xyz_min, xyz_max, world_size, near_clip, use_fourier_grid)
    
    # Initialize act_shift
    act_shift = torch.log(1.0 / alpha_init - 1.0)
    
    return density_grid, k0_grid, mlp, mask_cache, act_shift

def mask_near_camera(density_grid, xyz_min, xyz_max, world_size, near_clip, use_fourier_grid):
    """
    Mask out voxels too close to camera origins.
    """
    # Get grid coordinates
    grid_coords = get_grid_coordinates(world_size, xyz_min, xyz_max)
    
    # Assume camera origins are available in a list `camera_origins`
    # Split into batches of 100 for memory efficiency
    camera_origins = get_camera_origins() # Placeholder for actual camera data
    
    for batch_start in range(0, len(camera_origins), 100):
        batch_origins = camera_origins[batch_start:batch_start+100]
        
        for origin in batch_origins:
            if not use_fourier_grid:
                # Compute nearest distance in raw coordinates
                dists = compute_distances(grid_coords, origin)
                mask = dists <= near_clip
                density_grid[mask] = -100.0
            else:
                # Compute nearest distance in embedded space
                embedded_coords = apply_nerf_pos(grid_coords, fourier_freq_num)
                embedded_origins = apply_nerf_pos(origin.unsqueeze(0), fourier_freq_num).expand_as(embedded_coords)
                dists = torch.norm(embedded_coords - embedded_origins, dim=-1)
                mask = dists <= near_clip
                density_grid[mask] = -100.0

def update_mask_cache(density_grid, mask_cache, alpha_init, fast_color_thres):
    """
    Update occupancy cache using max pooling on alpha.
    """
    act_shift = torch.log(1.0 / alpha_init - 1.0)
    density = density_grid + act_shift
    alpha = 1.0 - torch.exp(-density) # Assuming exponential activation for alpha
    
    # Apply 3D max pooling
    alpha_pooled = F.max_pool3d(alpha.unsqueeze(0).unsqueeze(0), kernel_size=3, stride=1, padding=1).squeeze()
    
    # Update mask
    new_mask = alpha_pooled > fast_color_thres
    mask_cache = mask_cache & new_mask
    
    return mask_cache

def render_ray(ray_origin, ray_direction, density_grid, k0_grid, mlp, mask_cache, act_shift, xyz_min, xyz_max, world_size):
    """
    Render a single ray using direct voxel grid optimization.
    """
    # Sample points along the ray
    t_vals = torch.linspace(0, 1, num_samples)
    z_vals = ray_origin + ray_direction * t_vals
    
    # Check if points are within bounds and occupied
    # This step is simplified; actual implementation involves marching through occupied voxels
    
    # Query density and color
    densities = query_density(z_vals, density_grid, xyz_min, xyz_max, world_size)
    alphas = 1.0 - torch.exp(-(densities + act_shift))
    
    # Query color features
    colors = query_color(z_vals, k0_grid, xyz_min, xyz_max, world_size, mlp, ray_direction)
    
    # Composite color
    weights = alphas * torch.cumprod(torch.cat([torch.ones_like(alphas[:, :1]), 1. - alphas + 1e-10], -1), -1)[:, :-1]
    final_color = torch.sum(weights * colors, dim=-2)
    
    return final_color
```

## Operating characteristics
- **Time Complexity:** Training is significantly faster than hierarchical methods due to the single-grid structure and efficient occupancy caching [S1, unofficial].
- **Memory Complexity:** Memory usage is dominated by the single voxel grid. Near-camera masking requires batching camera origins to prevent excessive memory consumption [S1, unofficial].
- **Performance:** Achieves super-fast convergence for radiance field reconstruction [S1, unofficial].
- **Strengths:** Simple implementation, fast training, suitable for unbounded scenes with near-camera masking.
- **Limitations:** Single resolution may not capture fine details as well as hierarchical methods without scaling. Memory usage scales with the cube of the world size.
- **When to use:** When fast training convergence is prioritized over multi-scale detail capture, or for unbounded scenes where hierarchical structures are complex to manage.

## Background
DirectVoxGO was introduced by Cheng Sun, Min Sun, and Hwann-Tzong Chen in 2022 [S1, unofficial]. It is part of the family of direct voxel grid optimization methods, aiming to simplify the training process of neural radiance fields by optimizing a single voxel grid directly. The method builds on earlier work in voxel-based rendering and differentiable volume rendering.

## Sources
[S1] sjtuytc/UnboundedNeRFPytorch, https://github.com/sjtuytc/UnboundedNeRFPytorch, fetched 2026-10-26 (unofficial)
