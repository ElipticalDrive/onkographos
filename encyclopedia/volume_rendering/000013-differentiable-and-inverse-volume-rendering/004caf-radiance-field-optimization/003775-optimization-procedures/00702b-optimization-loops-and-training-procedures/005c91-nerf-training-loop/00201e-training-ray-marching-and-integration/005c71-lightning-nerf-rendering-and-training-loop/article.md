# Lightning-NeRF Rendering and Training Loop

Also called: Lightning NeRF

## Overview
Lightning-NeRF is a neural rendering method that accelerates training and inference by combining a hybrid density representation (voxel grid with hash encoding) with a decomposed view-independent/view-dependent MLP architecture [S1, unofficial]. It distinguishes itself from standard NeRF implementations by employing adaptive sampling warmup schedules and dynamic occupancy grid updates to prune empty space, enabling efficient training on large-scale real-world scenes [S1, unofficial].

## How it works
**Inputs:**
- Camera poses and intrinsics.
- Ground truth RGB images.
- Optional point cloud file (`.ply`) for initialization [S1, unofficial].

**Outputs:**
- Rendered RGB image.
- Accumulated opacity (accumulation).
- Depth map.
- Number of samples per ray.
- Optional residual RGB loss [S1, unofficial].

**Data Structures:**
- **Occupancy Grid:** A multi-resolution hash grid storing occupancy probabilities, updated dynamically based on density thresholds [S1, unofficial].
- **Density Grid:** A hybrid structure using a base voxel grid and hash encoding for density values [S1, unofficial].
- **Color Grid:** A multi-resolution hash grid for color features [S1, unofficial].
- **MLPs:** Two separate networks: View-Independent (VI) MLP encodes spatial position to intermediate features, and View-Dependent (VD) MLP encodes viewing direction and VI features to final RGB color [S1, unofficial].

**Parameters:**
- `near_plane`: Near clipping plane. Default: 0.01 [S1, unofficial].
- `far_plane`: Far clipping plane. Default: 6.0 [S1, unofficial].
- `alpha_thre`: Alpha threshold for occupancy culling. Default: 0.01 [S1, unofficial].
- `occ_num_samples_per_ray`: Coarse samples per ray. Default: 750 [S1, unofficial].
- `pdf_num_samples_per_ray`: Fine samples per ray. Default: 8 [S1, unofficial].
- `pdf_samples_warmup_step`: Steps to keep max fine samples. Default: 500 (code) / 100,000 (notes) [S1, unofficial].
- `pdf_samples_fixed_step`: Steps after which fine sampling ratio stabilizes. Default: 2,000 (code) / 100,000 (notes) [S1, unofficial].
- `pdf_samples_fixed_ratio`: Final ratio of fine samples. Default: 0.5 [S1, unofficial].
- `occ_grid_update_warmup_step`: Steps before occupancy grid updates. Not given in the sources.
- `bg_color`: Background color strategy. Options: "black" ([0,0,0]), "white" ([1,1,1]), "random" (sampled at runtime) [S1, unofficial].
- `init_density_value`: Initial density for pretraining. Default: 10.0 [S1, unofficial].
- `frontal_axis`: Axis for background augmentation. Default: "x" [S1, unofficial].
- `vi_mlp_num_layers`: Number of layers in VI MLP. Default: 3 [S1, unofficial].
- `vi_mlp_hidden_size`: Hidden size of VI MLP. Default: 64 [S1, unofficial].
- `vd_mlp_num_layers`: Number of layers in VD MLP. Default: 2 [S1, unofficial].
- `vd_mlp_hidden_size`: Hidden size of VD MLP. Default: 32 [S1, unofficial].
- `appearance_embedding_dim`: Dimension of appearance embedding. Default: 0 [S1, unofficial].
- `cone_angle`: Cone angle for sampling. Default: 0.004 [S1, unofficial].
- `density_grid_base_res`: Base resolution for density grid. Default: 256 [S1, unofficial].
- `density_log2_hashmap_size`: Log2 size of density hash map. Default: 24 [S1, unofficial].
- `color_grid_base_res`: Base resolution for color grid. Default: 128 [S1, unofficial].
- `color_grid_max_res`: Max resolution for color grid. Default: 2048 [S1, unofficial].
- `color_grid_fpl`: Features per level for color grid. Default: 2 [S1, unofficial].
- `color_log2_hashmap_size`: Log2 size of color hash map. Default: 19 [S1, unofficial].
- `color_grid_num_levels`: Number of levels for color grid. Default: 8 [S1, unofficial].
- `bg_density_grid_res`: Resolution for background density grid. Default: 32 [S1, unofficial].
- `bg_density_log2_hashmap_size`: Log2 size of background density hash map. Default: 18 [S1, unofficial].
- `bg_color_grid_base_res`: Base resolution for background color grid. Default: 32 [S1, unofficial].
- `bg_color_grid_max_res`: Max resolution for background color grid. Default: 128 [S1, unofficial].
- `bg_color_log2_hashmap_size`: Log2 size of background color hash map. Default: 16 [S1, unofficial].
- `occ_grid_base_res`: Base resolution for occupancy grid. Default: 256 [S1, unofficial].
- `occ_grid_num_levels`: Number of levels for occupancy grid. Default: 2 [S1, unofficial].
- `rgb_padding`: Padding for RGB loss. Default: None [S1, unofficial].
- `loss_coefficients`: Loss weights. Default: `{"rgb_loss": 1.0, "res_rgb_loss": 0.01}` [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    - Initialize the occupancy grid and sampler [S1, unofficial].
    - Initialize the collider with `near_plane` and `far_plane` [S1, unofficial].
    - If a `point_cloud_path` is provided:
        - Load vertices from the `.ply` file [S1, unofficial].
        - Transform vertices to normalized space using metadata transformation and scale [S1, unofficial].
        - Split vertices into `pretrain_fg_vertices` (inside the scene bounding box) and `pretrain_bg_vertices` (outside) [S1, unofficial].
        - Generate `pretrain_aug_vertices` by sampling points on the surfaces of the background bounding box, offset by 0.05 units [S1, unofficial].

2.  **Pretraining (Optional but typical):**
    - **Foreground Density:** Optimize the density encoding to match `init_density_value` at `pretrain_fg_vertices` locations for 20 iterations. Update the occupancy grid every iteration using `density_fn(x) * render_step_size` [S1, unofficial].
    - **Background Density:** Optimize the background density encoding to match `init_density_value` at `pretrain_bg_vertices` and `pretrain_aug_vertices` locations for 40 iterations [S1, unofficial].
    - Set `grid_initialized = True` [S1, unofficial].

3.  **Training Loop Callbacks:**
    - **Step Increment:** Increment `training_step` after each iteration [S1, unofficial].
    - **Occupancy Update:** Before each training iteration, update the occupancy grid using `occ_eval_fn = lambda x: field.density_fn(x) * render_step_size` and threshold `alpha_thre` [S1, unofficial].

4.  **Ray Marching and Integration (Forward Pass):**
    - **Determine Fine Samples:** Calculate `num_fine_samples` based on `training_step` [S1, unofficial]:
        - If `training_step < pdf_samples_warmup_step`: `num_fine_samples = pdf_num_samples_per_ray`.
        - If `pdf_samples_warmup_step <= training_step < pdf_samples_fixed_step`:
          $$ \text{ratio} = \frac{\text{training\_step} - \text{pdf\_samples\_warmup\_step}}{\text{pdf\_samples\_fixed\_step} - \text{pdf\_samples\_warmup\_step}} $$
          $$ \text{max\_ratio} = 1 - \text{pdf\_samples\_fixed\_ratio} $$
          $$ \text{num\_fine\_samples} = \text{round}(\text{pdf\_num\_samples\_per\_ray} \times (1 - \text{max\_ratio} \times \text{ratio})) $$
        - If `training_step >= pdf_samples_fixed_step`:
          $$ \text{num\_fine\_samples} = \text{round}(\text{pdf\_num\_samples\_per\_ray} \times \text{pdf\_samples\_fixed\_ratio}) $$
    - **Sampling:** Sample rays using the sampler with `ray_bundle`, `near_plane`, `far_plane`, `render_step_size`, `alpha_thre`, `cone_angle`, and `num_fine_samples`. Returns `ray_samples` and `ray_indices` [S1, unofficial].
    - **Field Query:** Query the neural field with `ray_samples` to obtain densities ($\sigma$) and colors ($C$) [S1, unofficial].
    - **Weight Computation:** Compute ray weights from densities:
        $$ w_i = \sigma_i \Delta t_i \exp\left(-\sum_{j=1}^{i-1} \sigma_j \Delta t_j\right) $$
        Implemented via `nerfacc.render_weight_from_density` using frustum starts/ends and packed info [S1, unofficial].
    - **Accumulation:**
        - **RGB:** $C_{comp} = \sum w_i C_i$ using `nerfacc.accumulate_along_rays`.
        - **Opacity:** $A = \sum w_i$ using `nerfacc.accumulate_along_rays` with `values=None`.
    - **Background Compositing:**
        - Determine background color $C_{bg}$:
            - If `bg_color` is "black": $[0, 0, 0]$.
            - If `bg_color` is "white": $[1, 1, 1]$.
            - If `bg_color` is "random": Sample $C_{bg} \sim \text{Uniform}(0, 1)$ per pixel [S1, unofficial].
        - Final RGB: $C_{final} = C_{comp} + C_{bg} (1 - A)$ [S1, unofficial].
        - If not training, clamp $C_{final}$ to $[0, 1]$ [S1, unofficial].
    - **Depth Estimation:**
        - Compute step centers: $t_i = \frac{t_{start, i} + t_{end, i}}{2}$.
        - Weighted sum: $D = \frac{\sum w_i t_i}{A + \epsilon}$, where $\epsilon = 10^{-10}$.
        - Clip depth to $[t_{\min}, t_{\max}]$ where $t_{\min}$ and $t_{\max}$ are the minimum and maximum step centers along the ray [S1, unofficial].

5.  **Loss and Metrics:**
    - Compute PSNR between $C_{final}$ and ground truth image [S1, unofficial].
    - If residual RGB loss is enabled, compute mean residual and add to outputs dictionary [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lightning_nerf_render_and_train(config, ray_bundle, gt_image, training_step):
    # 1. Determine number of fine samples based on warmup schedule
    if training_step < config.pdf_samples_warmup_step:
        num_fine_samples = config.pdf_num_samples_per_ray
    elif training_step < config.pdf_samples_fixed_step:
        ratio = (training_step - config.pdf_samples_warmup_step) / \
                (config.pdf_samples_fixed_step - config.pdf_samples_warmup_step)
        max_ratio = 1.0 - config.pdf_samples_fixed_ratio
        num_fine_samples = round(config.pdf_num_samples_per_ray * (1.0 - max_ratio * ratio))
    else:
        num_fine_samples = round(config.pdf_num_samples_per_ray * config.pdf_samples_fixed_ratio)

    # 2. Sample rays (occupancy grid pruning handled internally)
    ray_samples, ray_indices = sample_rays(ray_bundle, config.near_plane, config.far_plane,
                                           config.render_step_size, config.alpha_thre,
                                           config.cone_angle, num_fine_samples)

    # 3. Query Neural Field
    sigmas, colors = query_field(ray_samples)

    # 4. Pack ray indices for batched accumulation
    packed_info = pack_info(ray_indices, num_rays=len(ray_bundle))

    # 5. Compute weights from density
    t_starts = ray_samples.frustums.starts[..., 0]
    t_ends = ray_samples.frustums.ends[..., 0]
    deltas = t_ends - t_starts
    weights = compute_weights_from_density(t_starts, t_ends, sigmas[..., 0], packed_info)

    # 6. Accumulate RGB
    comp_rgb = accumulate_along_rays(weights, colors, ray_indices, len(ray_bundle))

    # 7. Accumulate Opacity
    accumulation = accumulate_along_rays(weights, None, ray_indices, len(ray_bundle))

    # 8. Composite with Background
    if config.background_color == "black":
        bg_color = torch.zeros_like(comp_rgb)
    elif config.background_color == "white":
        bg_color = torch.ones_like(comp_rgb)
    else:
        bg_color = torch.rand_like(comp_rgb)

    rgb = comp_rgb + bg_color * (1.0 - accumulation)

    if not training_step:
        rgb = torch.clamp(rgb, 0.0, 1.0)

    # 9. Compute Depth
    steps = (ray_samples.frustums.starts + ray_samples.frustums.ends) / 2.0
    depth = accumulate_along_rays(weights, steps, ray_indices, len(ray_bundle))
    depth = depth / (accumulation + 1e-10)
    depth = torch.clamp(depth, steps.min(), steps.max())

    # 10. Return Outputs
    outputs = {
        'rgb': rgb,
        'accumulation': accumulation,
        'depth': depth,
        'num_samples_per_ray': ray_samples.num_samples
    }

    if training_step and hasattr(config, 'res_rgb') and config.res_rgb is not None:
        outputs['res_rgb_loss'] = torch.mean(torch.abs(outputs['rgb'] - config.res_rgb))

    return outputs

def compute_weights_from_density(t_starts, t_ends, sigmas, packed_info):
    deltas = t_ends - t_starts
    exp_term = torch.exp(-torch.cumsum(sigmas * deltas, dim=-1))
    weights = sigmas * deltas * torch.cat([torch.ones_like(exp_term[..., :1]), exp_term[..., :-1]], dim=-1)
    return weights

def accumulate_along_rays(weights, values, ray_indices, n_rays):
    if values is None:
        values = torch.ones_like(weights)
    acc = torch.zeros(n_rays, *values.shape[1:], device=values.device)
    for i in range(n_rays):
        mask = ray_indices == i
        acc[i] = torch.sum(weights[mask] * values[mask], dim=0)
    return acc

def pack_info(ray_indices, num_rays):
    counts = torch.bincount(ray_indices, minlength=num_rays)
    return counts

def sample_rays(ray_bundle, near_plane, far_plane, render_step_size, alpha_thre, cone_angle, num_fine_samples):
    # Ray sampling with occupancy grid pruning. Implementation details not provided in sources.
    pass
```

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of rays and samples per ray. The hash encoding lookup is approximately $O(1)$ average case.
- **Memory Complexity:** Dominated by the hash grids for density and color. The size is determined by `density_log2_hashmap_size` (24) and `color_log2_hashmap_size` (19), allowing for high-resolution representations in limited memory [S1, unofficial].
- **Performance:** Significantly faster training than standard NeRF due to occupancy grid pruning and adaptive sampling.
- **Strengths:** Efficient training on large-scale real-world datasets (KITTI-360, Argoverse2). Good generalization with point cloud initialization.
- **Limitations:** Performance depends heavily on the quality of the input point cloud. The adaptive sampling schedule parameters require tuning for different scenes.
- **When to use:** Large-scale outdoor or indoor scenes where standard NeRF is too slow or memory-intensive.
- **When not to use:** Small-scale synthetic scenes (e.g., Blender) where `cone_angle` should be 0.0 and simpler representations may suffice [S1, unofficial].

## Background
Lightning-NeRF builds upon the principles of Instant-NGP (hash encoding) and NeRF (volume rendering) but introduces specific optimizations for real-world data, including point cloud initialization and background augmentation. The method is implemented in the Lightning-NeRF repository [S1, unofficial].

## Sources
[S1] Lightning-NeRF, https://github.com/VISION-SJTU/Lightning-NeRF, fetched 2026-10-29 (unofficial)
