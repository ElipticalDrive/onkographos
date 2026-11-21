# DirectVoxGO

Also called: Direct Voxel Grid Optimization, DVGO

## Overview
DirectVoxGO is a differentiable volume rendering method that optimizes a single 3D voxel grid to represent a Neural Radiance Field (NeRF), storing density and color features directly in grid tensors [S1, unofficial]. It solves the slow convergence and high memory overhead of MLP-based NeRFs by enabling direct optimization of grid values [S1, unofficial]. It distinguishes itself from standard NeRF by replacing coordinate-to-feature MLP lookups with direct grid sampling, and from sparse voxel methods by using a dense grid with occupancy caching and near-camera masking to handle unbounded scenes efficiently [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of calibrated images with known camera poses, a 3D world bounding box defined by `xyz_min` and `xyz_max`, and configuration parameters for grid resolution and feature dimensions [S1, unofficial].
*   **Outputs:** Optimized density and color feature voxel grids, and rendered images or depth maps for novel views [S1, unofficial].

**Data Structures:**
1.  **Density Grid:** A 3D tensor storing density values with `channels=1` [S1, unofficial]. If `use_fourier_grid` is true, it is a `FourierGrid` object; otherwise, it is a standard dense grid [S1, unofficial].
2.  **Color Feature Grid (k0):** A 3D tensor storing color features with `channels=k0_dim` [S1, unofficial].
3.  **Mask Cache:** A 3D boolean tensor indicating occupied voxels to skip rendering [S1, unofficial].
4.  **MLP (Optional):** A multi-layer perceptron used in the fine stage to map color features to RGB if `rgbnet_dim > 0` [S1, unofficial].

**Numbered Steps:**
1.  **Initialize Density Grid:** If `use_fourier_grid` is false, create a grid with `density_type`, `channels=1`, `world_size`, `xyz_min`, `xyz_max`, and `density_config` [S1, unofficial]. If true, create a `FourierGrid` using `FourierGrid_grid.create_grid` with `use_nerf_pos=True` and `fourier_freq_num` [S1, unofficial].
2.  **Initialize Color Feature Grid (k0):** If `rgbnet_dim <= 0` (coarse stage), set `k0_dim=3` and create a grid with `channels=3`; no MLP is used and grid values are treated directly as RGB [S1, unofficial]. If `rgbnet_dim > 0` (fine stage), set `k0_dim` based on `rgbnet_dim` and flags, create a grid with `channels=k0_dim`, and initialize an MLP (`rgbnet`) [S1, unofficial].
3.  **Initialize MLP Input Dimension (dim0):** Start with `dim0 = 3 + 3 * viewbase_pe * 2` [S1, unofficial]. If `rgbnet_direct` is true, add `k0_dim` [S1, unofficial]. If `rgbnet_direct` is false and `not full_implicit`, add `k0_dim - 3` [S1, unofficial]. Initialize MLP weights with ReLU activations and zero bias on the final layer [S1, unofficial].
4.  **Initialize Mask Cache:** If `mask_cache_path` is provided, load the `MaskGrid` and compute the mask for grid points [S1, unofficial]. Otherwise, create a mask of all `True` values [S1, unofficial]. Store as `grid.MaskGrid` with `xyz_min` and `xyz_max` [S1, unofficial].
5.  **Mask Out Near-Camera Voxels:** Non-FourierGrid: Compute grid XYZ coordinates. For each camera origin (batched into groups of 100 for memory efficiency), compute the nearest distance from grid points to the camera origin. Set density grid values to `-100` where `nearest_dist <= near_clip` [S1, unofficial]. FourierGrid: Compute normalized coordinates and apply NeRF position embedding. For each position embedding dimension, compute the nearest distance from grid points to the camera origin in the embedded space. Set density grid values to `-100` where `nearest_dist <= near_clip` [S1, unofficial].
6.  **Scale Volume Grid:** Update `world_size` and `voxel_size` [S1, unofficial]. Scale density and k0 grids [S1, unofficial]. If `total_voxels <= 256^3`, compute alpha from density, apply 3D max pooling (kernel=3, padding=1, stride=1), and update `mask_cache` by combining with `alpha > fast_color_thres` [S1, unofficial].
7.  **Update Occupancy Cache:** Compute grid XYZ for `mask_cache` resolution [S1, unofficial]. Query density and activate to get alpha [S1, unofficial]. Apply 3D max pooling (kernel=3, padding=1, stride=1) to alpha values [S1, unofficial]. Update `mask_cache` mask by ANDing with `alpha > fast_color_thres` [S1, unofficial].

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
*   `xyz_min`: Lower bound of the 3D world coordinate system (torch.Tensor buffer) [S1, unofficial].
*   `xyz_max`: Upper bound of the 3D world coordinate system (torch.Tensor buffer) [S1, unofficial].
*   `num_voxels`: Total number of voxels in the primary grid resolution [S1, unofficial].
*   `num_voxels_base`: Reference voxel count used to calculate base voxel size for resolution scaling [S1, unofficial].
*   `alpha_init`: Initial alpha value used to compute the density bias shift for the activation function [S1, unofficial].
*   `mask_cache_thres`: Threshold value for the mask cache to determine occupancy [S1, unofficial].
*   `fast_color_thres`: Threshold for determining if a voxel is opaque enough to skip color network evaluation [S1, unofficial].
*   `density_type`: String specifying the type of density grid (e.g., 'DenseGrid') [S1, unofficial].
*   `k0_type`: String specifying the type of color feature grid [S1, unofficial].
*   `rgbnet_dim`: Dimension of the color feature grid. If `<= 0`, uses coarse stage with 3-channel grid and no MLP; otherwise uses fine stage with feature grid and MLP [S1, unofficial].
*   `rgbnet_direct`: Boolean flag for direct color encoding in the MLP input [S1, unofficial].
*   `rgbnet_full_implicit`: Boolean flag for full implicit color encoding [S1, unofficial].
*   `rgbnet_depth`: Depth of the color MLP (number of layers) [S1, unofficial].
*   `rgbnet_width`: Width (hidden dimension) of the color MLP [S1, unofficial].
*   `viewbase_pe`: Number of positional encoding frequencies for view direction [S1, unofficial].

**Pitfalls:**
*   **Memory Usage:** Near-camera masking requires computing distances for all grid points against all camera origins. Camera origins must be split into batches (e.g., 100) to prevent OOM errors [S1, unofficial].
*   **FourierGrid Masking:** Uses position embedding dimensions for distance calculation instead of raw coordinates, requiring iteration over `pos_embed_output_dim` [S1, unofficial].
*   **Max Pooling:** 3D max pooling (kernel 3, padding 1, stride 1) is applied to alpha values to smooth occupancy. This prevents thin structures from being missed but may slightly blur fine details [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_world_size(xyz_min, xyz_max, voxel_size):
    return [math.floor((xyz_max[i] - xyz_min[i]) / voxel_size) for i in range(3)]

def compute_voxel_size(volume, num_voxels):
    return math.pow(volume / num_voxels, 1.0/3.0)

def max_pool_3d_alpha(alpha_grid, world_size, kernel=3, padding=1, stride=1):
    new_size = [math.ceil((world_size[i] + 2 * padding - kernel) / stride) + 1 for i in range(3)]
    pooled = [[[0.0 for _ in range(new_size[2])] for _ in range(new_size[1])] for _ in range(new_size[0])]
    for x in range(new_size[0]):
        for y in range(new_size[1]):
            for z in range(new_size[2]):
                max_val = 0.0
                for dx in range(kernel):
                    for dy in range(kernel):
                        for dz in range(kernel):
                            ox = x * stride + dx - padding
                            oy = y * stride + dy - padding
                            oz = z * stride + dz - padding
                            if 0 <= ox < len(alpha_grid) and 0 <= oy < len(alpha_grid[0]) and 0 <= oz < len(alpha_grid[0][0]):
                                max_val = max(max_val, alpha_grid[ox][oy][oz])
                pooled[x][y][z] = max_val
    return pooled

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
        
        volume = 1.0
        for i in range(3):
            volume *= (xyz_max[i] - xyz_min[i])
        voxel_size = compute_voxel_size(volume, num_voxels)
        self.world_size = compute_world_size(xyz_min, xyz_max, voxel_size)
        
        # Initialize Density Grid
        if not use_fourier_grid:
            self.density_grid = [[[0.0 for _ in range(self.world_size[2])] for _ in range(self.world_size[1])] for _ in range(self.world_size[0])]
        else:
            # FourierGrid handles positional encoding internally
            self.density_grid = [[[0.0 for _ in range(self.world_size[2])] for _ in range(self.world_size[1])] for _ in range(self.world_size[0])]
            
        # Initialize Color Feature Grid (k0)
        if rgbnet_dim <= 0:
            self.k0_dim = 3
        else:
            self.k0_dim = rgbnet_dim
        self.k0_grid = [[[0.0 for _ in range(self.k0_dim)] for _ in range(self.world_size[1])] for _ in range(self.world_size[0])]
        
        # Initialize MLP if needed
        self.rgbnet_weights = None
        if rgbnet_dim > 0:
            dim0 = 3 + 3 * viewbase_pe * 2
            if rgbnet_direct:
                dim0 += self.k0_dim
            else:
                dim0 += self.k0_dim - 3
            # [G] Standard MLP initialization with ReLU and zero bias on final layer
            self.rgbnet_weights = {'in_dim': dim0, 'hidden_dim': 64, 'out_dim': 3}
            
        # Initialize Mask Cache
        self.mask_cache = [[[True for _ in range(self.world_size[2])] for _ in range(self.world_size[1])] for _ in range(self.world_size[0])]
        self.mask_near_camera()

    def mask_near_camera(self, camera_origins):
        # Batches camera origins into groups of 100 for memory efficiency
        batch_size = 100
        for batch_start in range(0, len(camera_origins), batch_size):
            batch_end = min(batch_start + batch_size, len(camera_origins))
            for cam_idx in range(batch_start, batch_end):
                origin = camera_origins[cam_idx]
                for x in range(self.world_size[0]):
                    for y in range(self.world_size[1]):
                        for z in range(self.world_size[2]):
                            gx = self.xyz_min[0] + (x + 0.5) * (self.xyz_max[0] - self.xyz_min[0]) / self.world_size[0]
                            gy = self.xyz_min[1] + (y + 0.5) * (self.xyz_max[1] - self.xyz_min[1]) / self.world_size[1]
                            gz = self.xyz_min[2] + (z + 0.5) * (self.xyz_max[2] - self.xyz_min[2]) / self.world_size[2]
                            dist = math.sqrt((gx - origin[0])**2 + (gy - origin[1])**2 + (gz - origin[2])**2)
                            if dist <= self.near_clip:
                                self.density_grid[x][y][z] = -100.0

    def scale_volume_grid(self, num_voxels_base):
        voxel_size_base = compute_voxel_size(1.0, num_voxels_base)
        voxel_size = compute_voxel_size(1.0, self.num_voxels)
        self.world_size = compute_world_size(self.xyz_min, self.xyz_max, voxel_size)
        # Scale density and k0 grids
        # [G] Resampling logic omitted for brevity, standard trilinear/nearest interpolation applies
        if self.num_voxels <= 256**3:
            alpha_grid = self.compute_alpha_from_density()
            pooled_alpha = max_pool_3d_alpha(alpha_grid, self.world_size)
            self.update_occupancy_cache(pooled_alpha)

    def compute_alpha_from_density(self):
        act_shift = math.log(1.0 / self.alpha_init - 1)
        alpha_grid = [[[0.0 for _ in range(self.world_size[2])] for _ in range(self.world_size[1])] for _ in range(self.world_size[0])]
        for x in range(self.world_size[0]):
            for y in range(self.world_size[1]):
                for z in range(self.world_size[2]):
                    density = self.density_grid[x][y][z]
                    alpha = 1.0 - math.exp(-(density + act_shift))
                    alpha_grid[x][y][z] = max(0.0, alpha)
        return alpha_grid

    def update_occupancy_cache(self, alpha_grid):
        pooled_alpha = max_pool_3d_alpha(alpha_grid, self.world_size)
        for x in range(len(pooled_alpha)):
            for y in range(len(pooled_alpha[0])):
                for z in range(len(pooled_alpha[0][0])):
                    if pooled_alpha[x][y][z] <= self.fast_color_thres:
                        self.mask_cache[x][y][z] = False
```

## Subtopics

### [SPC-NeRF](12634-spc-nerf.md)
SPC-NeRF is a Rate-Distortion (RD) optimized predictive coding framework designed to compress Explicit Voxel Grid (EVG) represented Neural Radiance Fields (NeRFs) by removing spatial redundancy. It distinguishes itself from sibling methods like DirectVoxGO by introducing a two-stage finetuning process and a reference graph that jointly optimizes compression ratio and rendering distortion during training, achieving 32% bit savings compared to the state-of-the-art VQRF method.

### [GO-Surf](13553-go-surf.md)
GO-Surf is a direct feature grid optimization method for accurate and fast surface reconstruction from RGB-D sequences. It models the scene with a learned hierarchical feature voxel grid that encapsulates multi-level geometric and appearance local information. Unlike MLP-based methods, GO-Surf directly optimizes feature vectors in a voxel grid, which are decoded into signed distance field (SDF) and radiance values. This approach is approximately 60 times faster than MLP-based methods while preserving high-frequency details. It builds on the hierarchical grid optimization principles of [DirectVoxGO](8134-directvoxgo.md) but specifically targets surface reconstruction via SDF decoding and RGB-D supervision.

## Operating characteristics
*   **Time Complexity:** Training is significantly faster than MLP-based NeRFs because feature lookup is O(1) per sample, avoiding expensive MLP evaluations [S1, unofficial]. Convergence is super-fast due to direct optimization of grid values [S1, unofficial].
*   **Memory Complexity:** Memory usage is proportional to the number of voxels ($O(N^3)$) [S1, unofficial]. The dense grid can consume significant memory for high resolutions, mitigated by occupancy caching and near-camera masking [S1, unofficial].
*   **Measured Performance:** The method achieves super-fast convergence for radiance field reconstruction [S1, unofficial]. Specific benchmarks are not provided in the sources, but the method is cited for its speed advantage over MLP-based approaches [S1, unofficial].
*   **Scaling:** Scales with the number of voxels [S1, unofficial]. The method supports scaling the volume grid by adjusting `num_voxels` and `voxel_size` [S1, unofficial].
*   **Strengths:** Fast training, simple implementation, direct optimization of features [S1, unofficial].
*   **Limitations:** High memory usage for large scenes, dense grid representation may be inefficient for sparse scenes [S1, unofficial].
*   **When to use:** When fast training convergence is prioritized and memory is sufficient for the desired resolution [S1, unofficial].
*   **When not to use:** For very large, sparse scenes where sparse voxel methods are more memory-efficient [S1, unofficial].

## Background
DirectVoxGO was introduced by Cheng Sun, Min Sun, and Hwann-Tzong Chen in the paper "Direct voxel grid optimization: Super-fast convergence for radiance fields reconstruction" presented at the IEEE/CVF Conference on Computer Vision and Pattern Recognition (CVPR) in 2022 [S1, unofficial]. It builds upon the concept of explicit voxel grids for NeRF representation, aiming to overcome the slow convergence of implicit MLP-based methods [S1, unofficial]. The method is implemented in the GitHub repository `sjtuytc/UnboundedNeRFPytorch` [S1, unofficial].

## Sources
[S1] sjtuytc/UnboundedNeRFPytorch, https://github.com/sjtuytc/UnboundedNeRFPytorch, fetched 2026-10-26 (unofficial)
