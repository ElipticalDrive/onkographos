# DirectVoxGO

Also called: Direct Voxel Grid Optimization, DVGO

## Overview
DirectVoxGO is a differentiable volume rendering method that optimizes a single, dense 3D voxel grid to represent a Neural Radiance Field (NeRF), eliminating the need for a neural network to map coordinates to features [S1, unofficial]. It solves the slow convergence and high memory overhead of MLP-based NeRFs by storing density and color features directly in grid tensors, allowing for super-fast training convergence [S1, unofficial]. It distinguishes itself from standard NeRF by replacing the coordinate-to-feature MLP with direct grid lookups and from sparse voxel methods by using a dense grid with occupancy caching and near-camera masking to handle unbounded scenes efficiently [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of calibrated images with known camera poses (extrinsics and intrinsics), a 3D world bounding box defined by `xyz_min` and `xyz_max`, and configuration parameters for grid resolution and feature dimensions.
*   **Outputs:** Optimized density and color feature voxel grids, and rendered images or depth maps for novel views.

**Data Structures:**
1.  **Density Grid:** A 3D tensor of shape `(num_voxels_x, num_voxels_y, num_voxels_z, 1)` storing density values. If `use_fourier_grid` is true, this is a `FourierGrid` object; otherwise, it is a standard dense grid.
2.  **Color Feature Grid (k0):** A 3D tensor of shape `(num_voxels_x, num_voxels_y, num_voxels_z, k0_dim)` storing color features.
3.  **Mask Cache:** A 3D boolean tensor of shape `(mask_res_x, mask_res_y, mask_res_z)` indicating occupied voxels to skip rendering.
4.  **MLP (Optional):** A multi-layer perceptron used in the "fine" stage to map color features to RGB if `rgbnet_dim > 0`.

**Numbered Steps:**

1.  **Initialize Density Grid:**
    *   If `use_fourier_grid` is false: Create a grid with `density_type` (e.g., 'DenseGrid'), `channels=1`, `world_size`, `xyz_min`, `xyz_max`, and `density_config`.
    *   If `use_fourier_grid` is true: Create a `FourierGrid` using `FourierGrid_grid.create_grid` with `use_nerf_pos=True` and `fourier_freq_num`.

2.  **Initialize Color Feature Grid (k0):**
    *   If `rgbnet_dim <= 0` (coarse stage): Set `k0_dim = 3`. Create a grid with `channels=3`. No MLP is used; the grid values are directly treated as RGB.
    *   If `rgbnet_dim > 0` (fine stage): Set `k0_dim` based on `rgbnet_dim` and flags. Create a grid with `channels=k0_dim`. Initialize an MLP (`rgbnet`) to decode features.

3.  **Initialize MLP Input Dimension (dim0):**
    *   Start with `dim0 = 3 + 3 * viewbase_pe * 2` (RGB + view direction positional encodings).
    *   If `rgbnet_direct` is true: `dim0 += k0_dim`.
    *   If `rgbnet_direct` is false and `not full_implicit`: `dim0 += k0_dim - 3`.
    *   Initialize MLP weights with ReLU activations and zero bias on the final layer.

4.  **Initialize Mask Cache:**
    *   If `mask_cache_path` is provided: Load the `MaskGrid` and compute the mask for grid points.
    *   Otherwise: Create a mask of all `True` values.
    *   Store as `grid.MaskGrid` with `xyz_min` and `xyz_max`.

5.  **Mask Out Near-Camera Voxels:**
    *   **Non-FourierGrid:** Compute grid XYZ coordinates. For each camera origin (batched into groups of 100 for memory efficiency), compute the nearest distance from grid points to the camera origin. Set density grid values to `-100` where `nearest_dist <= near_clip`.
    *   **FourierGrid:** Compute normalized coordinates and apply NeRF position embedding. For each position embedding dimension, compute the nearest distance from grid points to the camera origin in the embedded space. Set density grid values to `-100` where `nearest_dist <= near_clip`.

6.  **Scale Volume Grid:**
    *   Update `world_size` and `voxel_size`.
    *   Scale density and k0 grids.
    *   If `total_voxels <= 256^3`: Compute alpha from density, apply 3D max pooling (kernel=3, padding=1, stride=1), and update `mask_cache` by combining with `alpha > fast_color_thres`.

7.  **Update Occupancy Cache:**
    *   Compute grid XYZ for `mask_cache` resolution.
    *   Query density and activate to get alpha.
    *   Apply 3D max pooling (kernel=3, padding=1, stride=1) to alpha values.
    *   Update `mask_cache` mask by ANDing with `alpha > fast_color_thres`.

**Formulas:**
*   Density activation shift:
    $$ \text{act\_shift} = \log\left(\frac{1}{\alpha_{\text{init}}} - 1\right) $$
*   Base voxel size:
    $$ \text{voxel\_size}_{\text{base}} = \sqrt[3]{\frac{\text{Volume}}{\text{num\_voxels}_{\text{base}}}} $$
*   Current voxel size:
    $$ \text{voxel\_size} = \sqrt[3]{\frac{\text{Volume}}{\text{num\_voxels}}} $$
*   World size (grid dimensions):
    $$ \text{world\_size} = \left\lfloor \frac{\text{xyz}_{\text{max}} - \text{xyz}_{\text{min}}}{\text{voxel\_size}} \right\rfloor $$
*   Voxel size ratio:
    $$ \text{voxel\_size}_{\text{ratio}} = \frac{\text{voxel\_size}}{\text{voxel\_size}_{\text{base}}} $$

**Parameters:**
*   `xyz_min`: Lower bound of the 3D world coordinate system (torch.Tensor buffer).
*   `xyz_max`: Upper bound of the 3D world coordinate system (torch.Tensor buffer).
*   `num_voxels`: Total number of voxels in the primary grid resolution.
*   `num_voxels_base`: Reference voxel count used to calculate base voxel size for resolution scaling.
*   `alpha_init`: Initial alpha value used to compute the density bias shift for the activation function.
*   `mask_cache_thres`: Threshold value for the mask cache to determine occupancy.
*   `fast_color_thres`: Threshold for determining if a voxel is opaque enough to skip color network evaluation.
*   `density_type`: String specifying the type of density grid (e.g., 'DenseGrid').
*   `k0_type`: String specifying the type of color feature grid.
*   `rgbnet_dim`: Dimension of the color feature grid. If `<= 0`, uses coarse stage with 3-channel grid and no MLP; otherwise uses fine stage with feature grid and MLP.
*   `rgbnet_direct`: Boolean flag for direct color encoding in the MLP input.
*   `rgbnet_full_implicit`: Boolean flag for full implicit color encoding.
*   `rgbnet_depth`: Depth of the color MLP (number of layers).
*   `rgbnet_width`: Width (hidden dimension) of the color MLP.
*   `viewbase_pe`: Number of positional encoding frequencies for view direction.

**Pitfalls:**
*   **Memory Usage:** Near-camera masking requires computing distances for all grid points against all camera origins. Camera origins must be split into batches (e.g., 100) to prevent OOM errors [S1, unofficial].
*   **FourierGrid Masking:** Uses position embedding dimensions for distance calculation instead of raw coordinates, requiring iteration over `pos_embed_output_dim` [S1, unofficial].
*   **Max Pooling:** 3D max pooling (kernel 3, padding 1, stride 1) is applied to alpha values to smooth occupancy. This prevents thin structures from being missed but may slightly blur fine details [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class DirectVoxGO:
    def __init__(self, xyz_min, xyz_max, num_voxels, alpha_init, 
                 rgbnet_dim, rgbnet_direct, viewbase_pe, 
                 use_fourier_grid=False, fourier_freq_num=4,
                 near_clip=0.0, fast_color_thres=0.0025):
        self.xyz_min = xyz_min
        self.xyz_max = xyz_max
        self.num_voxels = num_voxels
        self.alpha_init = alpha_init
        self.rgbnet_dim = rgbnet_dim
        self.rgbnet_direct = rgbnet_direct
        self.viewbase_pe = viewbase_pe
        self.use_fourier_grid = use_fourier_grid
        self.fourier_freq_num = fourier_freq_num
        self.near_clip = near_clip
        self.fast_color_thres = fast_color_thres
        
        # Calculate grid dimensions
        volume = torch.prod(xyz_max - xyz_min)
        voxel_size = torch.cbrt(volume / num_voxels)
        self.world_size = torch.floor((xyz_max - xyz_min) / voxel_size).long()
        
        # Initialize Density Grid
        if not use_fourier_grid:
            self.density_grid = nn.Parameter(torch.zeros(*self.world_size, 1))
        else:
            # FourierGrid implementation details omitted as per source constraints
            # [G] In practice, this would be a specialized grid class handling positional encoding
            self.density_grid = nn.Parameter(torch.zeros(*self.world_size, 1))
            
        # Initialize Color Feature Grid (k0)
        if rgbnet_dim <= 0:
            self.k0_dim = 3
        else:
            # k0_dim calculation depends on rgbnet_dim and flags
            self.k0_dim = rgbnet_dim 
            
        self.k0_grid = nn.Parameter(torch.zeros(*self.world_size, self.k0_dim))
        
        # Initialize MLP if needed
        self.rgbnet = None
        if rgbnet_dim > 0:
            dim0 = 3 + 3 * viewbase_pe * 2
            if rgbnet_direct:
                dim0 += self.k0_dim
            else:
                dim0 += self.k0_dim - 3
            
            layers = []
            # [G] Assuming standard MLP structure for color decoding
            # Source mentions ReLU activations and zero bias on final layer
            for i in range(3): # Assuming depth 3 as common default if not specified
                layers.append(nn.Linear(dim0 if i == 0 else 64, 64))
                layers.append(nn.ReLU())
            layers.append(nn.Linear(64, 3))
            layers[-1].bias.data.zero_()
            self.rgbnet = nn.Sequential(*layers)

        # Initialize Mask Cache
        self.mask_cache = torch.ones(*self.world_size, dtype=torch.bool)
        
        # Apply initial masking
        self.mask_near_camera()

    def mask_near_camera(self):
        # [G] Simplified logic for batching cameras
        # In practice, iterate over camera origins in batches of 100
        pass

    def forward(self, rays):
        # rays: [N, 6] (origin, direction)
        # Sample points along rays
        # Query density and color features from grids
        # Compute alpha and RGB
        # Return rendered color, alpha, depth
        pass

    def update_mask_cache(self):
        # Query density, compute alpha
        # Apply 3D max pooling
        # Update mask_cache
        pass
```

## Subtopics

### [SPC-NeRF](12634-spc-nerf.md)
SPC-NeRF is a Rate-Distortion (RD) optimized predictive coding framework designed to compress Explicit Voxel Grid (EVG) represented Neural Radiance Fields (NeRFs) by removing spatial redundancy. It distinguishes itself from sibling methods like DirectVoxGO by introducing a two-stage finetuning process and a reference graph that jointly optimizes compression ratio and rendering distortion during training, achieving 32% bit savings compared to the state-of-the-art VQRF method.

### [GO-Surf](13553-go-surf.md)
GO-Surf is a direct feature grid optimization method for accurate and fast surface reconstruction from RGB-D sequences. It models the scene with a learned hierarchical feature voxel grid that encapsulates multi-level geometric and appearance local information. Unlike MLP-based methods, GO-Surf directly optimizes feature vectors in a voxel grid, which are decoded into signed distance field (SDF) and radiance values. This approach is approximately 60 times faster than MLP-based methods while preserving high-frequency details. It builds on the hierarchical grid optimization principles of [DirectVoxGO](8134-directvoxgo.md) but specifically targets surface reconstruction via SDF decoding and RGB-D supervision.

## Operating characteristics
*   **Time Complexity:** Training is significantly faster than MLP-based NeRFs because feature lookup is O(1) per sample, avoiding expensive MLP evaluations. Convergence is super-fast due to direct optimization of grid values [S1, unofficial].
*   **Memory Complexity:** Memory usage is proportional to the number of voxels ($O(N^3)$). The dense grid can consume significant memory for high resolutions, mitigated by occupancy caching and near-camera masking [S1, unofficial].
*   **Measured Performance:** The method achieves super-fast convergence for radiance field reconstruction [S1, unofficial]. Specific benchmarks are not provided in the sources, but the method is cited for its speed advantage over MLP-based approaches [S1, unofficial].
*   **Scaling:** Scales with the number of voxels. The method supports scaling the volume grid by adjusting `num_voxels` and `voxel_size` [S1, unofficial].
*   **Strengths:** Fast training, simple implementation, direct optimization of features.
*   **Limitations:** High memory usage for large scenes, dense grid representation may be inefficient for sparse scenes.
*   **When to use:** When fast training convergence is prioritized and memory is sufficient for the desired resolution.
*   **When not to use:** For very large, sparse scenes where sparse voxel methods are more memory-efficient.

## Background
DirectVoxGO was introduced by Cheng Sun, Min Sun, and Hwann-Tzong Chen in the paper "Direct voxel grid optimization: Super-fast convergence for radiance fields reconstruction" presented at the IEEE/CVF Conference on Computer Vision and Pattern Recognition (CVPR) in 2022 [S1, unofficial]. It builds upon the concept of explicit voxel grids for NeRF representation, aiming to overcome the slow convergence of implicit MLP-based methods. The method is implemented in the GitHub repository `sjtuytc/UnboundedNeRFPytorch` [S1, unofficial].

## Sources
[S1] sjtuytc/UnboundedNeRFPytorch, https://github.com/sjtuytc/UnboundedNeRFPytorch, fetched 2026-10-26 (unofficial)
