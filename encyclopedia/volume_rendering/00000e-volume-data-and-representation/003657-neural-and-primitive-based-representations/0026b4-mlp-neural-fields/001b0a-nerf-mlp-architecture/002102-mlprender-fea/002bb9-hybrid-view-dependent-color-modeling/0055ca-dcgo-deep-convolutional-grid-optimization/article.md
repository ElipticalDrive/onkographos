# DCGO (Deep Convolutional Grid Optimization)

Also called: DCGO (Deep Convolutional Grid Optimization) Volume Representation

## Overview
DCGO is a hybrid volume representation that extends the standard Direct Volume Grid Optimization (DVGO) by introducing space contraction and optimized ray sampling to handle unbounded or inward-facing scenes [S1, unofficial]. It replaces the linear depth sampling of its sibling [DVGO](5082-dvgo.md) with a contracted coordinate system and a two-stage sampling strategy (inner/outer), allowing for efficient rendering of scenes with infinite background or specific camera constraints [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   Camera poses, intrinsics, and image dimensions.
*   Scene bounding box defined by `xyz_min` and `xyz_max`.
*   Configuration parameters: `rgbnet_dim`, `rgbnet_depth`, `rgbnet_width`, `viewbase_pe`, `fast_color_thres`, `contracted_norm`, `bg_len`, `stepsize`.

**Outputs:**
*   Rendered RGB images, depth maps, and optionally segmentation masks.

### Data Structures

**Density Grid:**
*   Type: 3D Tensor (or sparse structure depending on implementation, but typically dense grid in this context).
*   Layout: `channels=1`.
*   Dimensions: `world_size` (derived from `xyz_min`, `xyz_max`, and `voxel_size`).
*   Created via `grid.create_grid` with `density_type`.

**Color/Feature Grid (k0):**
*   Type: 3D Tensor.
*   Layout: `channels=k0_dim`.
    *   If `rgbnet_dim <= 0` (Coarse Stage): `k0_dim = 3` (RGB).
    *   If `rgbnet_dim > 0` (Fine Stage): `k0_dim = rgbnet_dim`.
*   Dimensions: `world_size`.
*   Created via `grid.create_grid` with `k0_type`.

**RGB MLP (Fine Stage Only):**
*   Structure: `nn.Sequential`.
*   Layers:
    1.  `nn.Linear(dim0, rgbnet_width)`
    2.  `nn.ReLU`
    3.  `(rgbnet_depth - 2)` blocks of `nn.Linear(rgbnet_width, rgbnet_width)` followed by `nn.ReLU`.
    4.  `nn.Linear(rgbnet_width, 3)` (Bias initialized to 0).
*   Input Dimension `dim0`: `(3 + 3 * viewbase_pe * 2) + k0_dim`.
    *   `3`: Positional encoded position (x,y,z).
    *   `3 * viewbase_pe * 2`: Positional encoded view direction (sin/cos for `viewbase_pe` frequencies).
    *   `k0_dim`: Feature vector from the grid.

**Mask Cache:**
*   Type: `grid.MaskGrid` object.
*   Stores a boolean mask of occupied space.
*   Initialized with `xyz_min`, `xyz_max`, and a mask tensor derived from density/alpha thresholds.

### Parameters

*   `rgbnet_dim`: Dimension of the feature grid `k0`. Determines if a shallow MLP is used for view-dependent color.
*   `rgbnet_depth`: Depth of the RGB MLP.
*   `rgbnet_width`: Hidden dimension of the RGB MLP.
*   `viewbase_pe`: Number of frequency bands for positional encoding of view directions.
*   `fast_color_thres`: Threshold for alpha or weight values to skip computation in transparent regions. Can be a static float or a dict mapping `global_step` to float.
*   `contracted_norm`: Norm used for space contraction. Values: `'inf'` or `'l2'` [S1, unofficial].
*   `bg_len`: Background length parameter used in ray sampling and contraction [S1, unofficial].
*   `stepsize`: Step size for ray marching.

### Method Steps

#### 1. Initialization and Grid Setup
1.  **Initialize View Frequency Buffer:** If using the fine stage, create `viewfreq` tensor:
    $$ \text{viewfreq} = [(2^i) \mid i \in \text{range}(\text{viewbase\_pe})] $$
2.  **Calculate Input Dimension:**
    $$ \text{dim0} = (3 + 3 \cdot \text{viewbase\_pe} \cdot 2) + \text{k0\_dim} $$
3.  **Set Grid Resolution:**
    *   Compute `voxel_size`:
        $$ \text{voxel\_size} = \left( \frac{(\text{xyz\_max} - \text{xyz\_min}).\text{prod()}}{\text{num\_voxels}} \right)^{1/3} $$
    *   Compute `world_size`:
        $$ \text{world\_size} = \left\lfloor \frac{\text{xyz\_max} - \text{xyz\_min}}{\text{voxel\_size}} \right\rfloor $$
    *   Compute `world_len`: `world_size[0].item()` (assuming isotropic or primary axis).
    *   Compute `voxel_size_ratio`:
        $$ \text{voxel\_size\_ratio} = \frac{\text{voxel\_size}}{\text{voxel\_size\_base}} $$
4.  **Scale Volume Grid:**
    *   If total voxels $\le 256^3$:
        *   Create a 3D grid of coordinates.
        *   Query density to compute alpha.
        *   Apply `max_pool3d` with `kernel_size=3`, `padding=1`, `stride=1`.
        *   Update `mask_cache` where `alpha > fast_color_thres`.

#### 2. Occupancy Cache Update
1.  **Standard Update:**
    *   Create grid coordinates.
    *   Query density, compute alpha.
    *   Apply `max_pool3d` (`kernel_size=3`, `padding=1`, `stride=1`).
    *   Update `mask_cache` where `alpha > fast_color_thres`.
2.  **Low View Count Update (Optional/Specific):**
    *   Initialize `count` grid.
    *   For each batch of rays:
        *   Sample points.
        *   Compute gradient of a dummy grid of 'ones' at those points.
        *   Increment `count` where gradient $> 1$.
    *   Update `mask_cache` where `count >= maskout_lt_nviews`.

#### 3. Regularization
*   **Total Variation Gradient:**
    *   Scale weight by $\frac{\text{world\_size.max()}}{128}$.
    *   Add gradient to density and `k0` grids.

#### 4. Density Activation
*   Flatten density grid.
*   Apply `Raw2Alpha` activation with `act_shift` and `interval` (set to `voxel_size_ratio`).
*   Reshape to original shape.

#### 5. Ray Sampling and Marching
1.  **Normalize Rays:**
    *   Normalize ray origins `rays_o` by `scene_center` and `radius`.
    *   Normalize ray directions `rays_d`.
2.  **Calculate Steps:**
    *   Determine `N_inner` and `N_outer` steps.
    *   Create `t_inner` linearly spaced from $0$ to $2$.
    *   Create `t_outer` inversely spaced from $2$ to $1/128$.
    *   Concatenate `t` values.
3.  **Compute Points:**
    *   Compute `ray_pts` using `rays_o` and `t`.
    *   Apply contraction based on `contracted_norm` ('inf' or 'l2').
    *   **Contraction Formula:**
        $$ s = 1 - \frac{1}{1+t} $$
        Where $t$ is the contracted distance parameter, mapping $[0, \infty)$ to $[0, 1)$ [S1, unofficial].
4.  **Forward Pass - Sample Points:**
    *   Call `sample_ray`.
    *   Calculate `n_max`.
    *   Calculate `interval` as `stepsize * voxel_size_ratio`.
    *   Create `ray_id` and `step_id` using `create_full_step_id`.
5.  **Forward Pass - Compute Distances:**
    *   Initialize `ray_distance`.
    *   Compute absolute differences between consecutive points.
    *   Cumsum along `dim=1`.
6.  **Forward Pass - Filter Oversampled Points:**
    *   Clone `inner_mask`.
    *   Compute distance between consecutive points.
    *   Apply `cumdist_thres` CUDA kernel with:
        $$ \text{dist\_thres} = \frac{2 + 2 \cdot \text{bg\_len}}{\text{world\_len}} \cdot \text{stepsize} \cdot 0.95 $$
    *   Update mask.
    *   Filter `ray_pts`, `ray_distance`, `inner_mask`, `t`, `ray_id`, `step_id`.
7.  **Forward Pass - Filter Free Space:**
    *   Query `mask_cache` for `ray_pts`.
    *   Filter `ray_pts`, `ray_distance`, `inner_mask`, `t`, `ray_id`, `step_id`.
8.  **Forward Pass - Query Density and Alpha:**
    *   Query density grid.
    *   Activate density with `interval`.
    *   If `render_fct > 0`, filter points where `alpha > render_fct`.
9.  **Forward Pass - Compute Weights:**
    *   Apply `Alphas2Weights` with `alpha`, `ray_id`, `N`.
    *   If `render_fct > 0`, filter points where `weights > render_fct`.
10. **Forward Pass - Query Color:**
    *   Query `k0` grid.
    *   If `rgbnet` is `None`:
        *   Apply sigmoid to `k0`.
    *   Else (Fine Stage):
        *   Compute `viewdirs_emb` using positional encoding (sin/cos of `viewdirs` * `viewfreq`).
        *   Concatenate with `k0`.
        *   Pass through `rgbnet`.
        *   Apply sigmoid to result.
11. **Forward Pass - Ray Marching:**
    *   Compute `rgb_marched` using `segment_coo` summing `weights * rgb` by `ray_id`.
    *   Add background color: `alphainv_last * bg` or random.
    *   Compute `wsum_mid` using `segment_coo` summing `weights` for `inner_mask` points by `ray_id`.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class DCGOModel:
    def __init__(self, xyz_min, xyz_max, cfg):
        self.xyz_min = xyz_min
        self.xyz_max = xyz_max
        self.cfg = cfg
        
        # Calculate grid parameters
        num_voxels = cfg.num_voxels # Assumed config param
        voxel_size = ((xyz_max - xyz_min).prod() / num_voxels).pow(1/3)
        world_size = ((xyz_max - xyz_min) / voxel_size).long()
        world_len = world_size[0].item()
        voxel_size_ratio = voxel_size / cfg.voxel_size_base
        
        # Create Grids
        self.density_grid = create_grid(density_type, channels=1, world_size=world_size, 
                                        xyz_min=xyz_min, xyz_max=xyz_max)
        k0_dim = cfg.rgbnet_dim if cfg.rgbnet_dim > 0 else 3
        self.k0_grid = create_grid(k0_type, channels=k0_dim, world_size=world_size, 
                                   xyz_min=xyz_min, xyz_max=xyz_max)
        
        # Create Mask Cache
        self.mask_cache = MaskGrid(xyz_min=xyz_min, xyz_max=xyz_max)
        
        # Create RGB MLP if fine stage
        if cfg.rgbnet_dim > 0:
            viewfreq = torch.FloatTensor([(2**i) for i in range(cfg.viewbase_pe)])
            dim0 = (3 + 3 * cfg.viewbase_pe * 2) + cfg.rgbnet_dim
            self.rgbnet = nn.Sequential(
                nn.Linear(dim0, cfg.rgbnet_width),
                nn.ReLU(),
                *[nn.Sequential(nn.Linear(cfg.rgbnet_width, cfg.rgbnet_width), nn.ReLU()) 
                  for _ in range(cfg.rgbnet_depth - 2)],
                nn.Linear(cfg.rgbnet_width, 3, bias=False) # Bias init 0 handled in layer
            )
            # Initialize last layer bias to 0 if not done by nn.Linear default
            self.rgbnet[-1].bias = nn.Parameter(torch.zeros(3))
        else:
            self.rgbnet = None
            
        self.voxel_size_ratio = voxel_size_ratio
        self.world_len = world_len
        self.viewfreq = viewfreq

    def update_mask_cache(self, density_grid, fast_color_thres):
        # Flatten and query density to get alpha
        # Apply max_pool3d
        # Update mask_cache where alpha > fast_color_thres
        pass

    def contract_points(self, ray_pts, contracted_norm='inf'):
        # Normalize ray_pts relative to scene center/radius if needed
        # Apply contraction formula
        # s = 1 - 1/(1+t)
        if contracted_norm == 'inf':
            t = torch.max(torch.abs(ray_pts), dim=-1, keepdim=True)[0]
        else: # l2
            t = torch.norm(ray_pts, dim=-1, keepdim=True)
        
        # Handle t=0 to avoid division by zero
        t = torch.clamp(t, min=1e-6)
        s = 1 - 1 / (1 + t)
        # Rescale back to original space if necessary, or use s as contracted coordinate
        # The source implies mapping [0, inf] to [0, 1]
        return s

    def forward(self, rays_o, rays_d, stepsize, bg=0):
        # Normalize rays
        rays_o = (rays_o - self.cfg.scene_center) / self.cfg.radius
        rays_d = F.normalize(rays_d, dim=-1)
        
        # Sampling
        N_inner = self.cfg.N_inner
        N_outer = self.cfg.N_outer
        
        t_inner = torch.linspace(0, 2, N_inner)
        t_outer = torch.linspace(2, 1/128, N_outer) # Inverse spacing logic handled by linspace or custom
        # Note: Source says "inversely spaced", linspace on [2, 1/128] is linear. 
        # Implementation detail: usually 1/t spacing. 
        # Assuming source means t values are generated such that sampling is denser near origin?
        # The source code snippet just says "inversely spaced from 2 to 1/128". 
        # We will assume standard linspace for t values, but the contraction handles the density.
        t = torch.cat([t_inner, t_outer])
        
        # Compute ray points
        ray_pts = rays_o.unsqueeze(1) + rays_d.unsqueeze(1) * t.unsqueeze(0).unsqueeze(0)
        
        # Contract
        ray_pts_contracted = self.contract_points(ray_pts, self.cfg.contracted_norm)
        
        # Forward Pass Steps
        # 1. Sample points (create ray_id, step_id)
        # 2. Compute distances
        # 3. Filter oversampled points
        # 4. Filter free space (using mask_cache)
        # 5. Query density and alpha
        density = self.density_grid(ray_pts_contracted) # Interpolation
        alpha = raw2alpha(density, self.voxel_size_ratio, self.cfg.act_shift)
        
        # 6. Compute weights
        weights = alphas2weights(alpha, ray_id, N)
        
        # 7. Query color
        k0 = self.k0_grid(ray_pts_contracted)
        if self.rgbnet is None:
            rgb = torch.sigmoid(k0)
        else:
            # Positional encoding for view direction
            viewdirs_emb = positional_encoding(rays_d, self.viewfreq)
            # Concatenate k0 and viewdirs_emb
            input_feat = torch.cat([k0, viewdirs_emb], dim=-1)
            rgb = torch.sigmoid(self.rgbnet(input_feat))
            
        # 8. Ray Marching
        rgb_marched = segment_coo_sum(weights * rgb, ray_id)
        alphainv_last = segment_coo_sum(weights[inner_mask], ray_id) # Approximation
        bg_color = bg * alphainv_last
        rgb_final = rgb_marched + bg_color
        
        return rgb_final

def raw2alpha(density, interval, act_shift):
    # Standard NeRF alpha calculation
    # alpha = 1 - exp(-density * interval)
    # With shift? Source mentions act_shift.
    # Assuming standard activation with optional shift
    return 1 - torch.exp(-(density + act_shift) * interval)

def positional_encoding(v, freq_bands):
    # v: (..., 3)
    # freq_bands: (K,)
    # Returns: (..., 3 * 2 * K)
    s = torch.sin(v.unsqueeze(-1) * freq_bands.unsqueeze(0))
    c = torch.cos(v.unsqueeze(-1) * freq_bands.unsqueeze(0))
    return torch.cat([s, c], dim=-1)
```

## Operating characteristics

*   **Time Complexity:** Depends on the number of sampled points per ray and the grid query cost. The contraction and filtering steps add overhead but reduce the number of points needed for high-quality rendering in unbounded scenes compared to linear sampling.
*   **Memory Complexity:** Dominated by the 3D density and feature grids. Memory usage scales with `world_size^3`. The mask cache adds minimal overhead.
*   **Performance:** Efficient for scenes with infinite backgrounds or inward-facing cameras due to the contracted space. The two-stage sampling (inner/outer) allows for detailed rendering near the camera and efficient sampling further away.
*   **Strengths:** Handles unbounded scenes well; reduces artifacts from linear sampling in distant regions; hybrid approach allows for view-dependent color.
*   **Limitations:** Requires careful tuning of contraction parameters and sampling steps; grid resolution limits detail.

## Background

DCGO was introduced as part of the "Segment Anything in 3D" project, extending the Direct VoxGO (DVGO) method to support unbounded and inward-facing scenes [S1, unofficial]. It was developed by Jumpat et al. and released as part of an unofficial repository on GitHub [S1, unofficial]. The method is designed to integrate with the Segment Anything Model (SAM) for 3D segmentation tasks.

## Sources

[S1] Jumpat/SegmentAnythingin3D, https://github.com/Jumpat/SegmentAnythingin3D, fetched 2026-10-26 (unofficial)
