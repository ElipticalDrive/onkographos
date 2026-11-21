# Lightning-NeRF Rendering and Training Loop

Also called: Lightning NeRF

## Overview
Lightning-NeRF is a neural rendering method that accelerates training and inference by combining a hybrid density representation (voxel grid with hash encoding) with a decomposed view-independent/view-dependent MLP architecture [S1, unofficial]. It distinguishes itself from standard NeRF implementations by employing adaptive sampling warmup schedules and dynamic occupancy grid updates to prune empty space, enabling efficient training on large-scale real-world scenes [S1, unofficial].

## How it works
**Inputs:**
- Camera poses and intrinsics.
- Ground truth RGB images.
- Optional point cloud file (`.ply`) for initialization [S1, unofficial].
- Configuration parameters (see Parameters).

**Outputs:**
- Rendered RGB image.
- Accumulated opacity (accumulation).
- Depth map.
- Number of samples per ray.
- Optional residual RGB loss [S1, unofficial].

**Data Structures:**
- **Occupancy Grid:** A multi-resolution hash grid storing occupancy probabilities. Updated dynamically based on density thresholds.
- **Density Grid:** A hybrid structure using a base voxel grid and hash encoding for density values.
- **Color Grid:** A multi-resolution hash grid for color features.
- **MLPs:** Two separate networks:
    - View-Independent (VI) MLP: Encodes spatial position to intermediate features.
    - View-Dependent (VD) MLP: Encodes viewing direction and VI features to final RGB color.

**Parameters:**
- `near_plane`: Near clipping plane. Default: 0.01 [S1, unofficial].
- `far_plane`: Far clipping plane. Default: 6.0 (KITTI-360), 10.0 (Argoverse2) [S1, unofficial].
- `alpha_thre`: Alpha threshold for occupancy culling. Default: 0.01 (KITTI-360), 0.02 (Argoverse2) [S1, unofficial].
- `occ_num_samples_per_ray`: Coarse samples per ray. Default: 750 [S1, unofficial].
- `pdf_num_samples_per_ray`: Fine samples per ray. Default: 8 [S1, unofficial].
- `pdf_samples_warmup_step`: Steps to keep max fine samples. Default: 100,000 (notes) / 500 (code) [S1, unofficial]. *Note: Sources disagree on this value. The notes cite 100,000, while the source code defines 500. Implementers should verify against the specific configuration file used.*
- `pdf_samples_fixed_step`: Steps after which fine sampling ratio stabilizes. Default: 100,000 (notes) / 2,000 (code) [S1, unofficial]. *Note: See discrepancy above.*
- `pdf_samples_fixed_ratio`: Final ratio of fine samples. Default: 0.5 [S1, unofficial].
- `occ_grid_update_warmup_step`: Steps before occupancy grid updates. Default: 256 (KITTI-360), 2 (Argoverse2) [S1, unofficial].
- `bg_color`: Background color strategy. Options: "black" ([0,0,0]), "white" ([1,1,1]), "random" (sampled at runtime) [S1, unofficial].
- `init_density_value`: Initial density for pretraining. Default: 10.0 [S1, unofficial].
- `frontal_axis`: Axis for background augmentation. Default: "x" [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    - Initialize the occupancy grid and sampler.
    - Initialize the collider with `near_plane` and `far_plane`.
    - If a `point_cloud_path` is provided:
        - Load vertices from the `.ply` file.
        - Transform vertices to normalized space using metadata transformation and scale.
        - Split vertices into `pretrain_fg_vertices` (inside the scene bounding box) and `pretrain_bg_vertices` (outside).
        - Generate `pretrain_aug_vertices` by sampling points on the surfaces of the background bounding box, offset by 0.05 units [S1, unofficial].

2.  **Pretraining (Optional but typical):**
    - **Foreground Density:** Optimize the density encoding to match `init_density_value` at `pretrain_fg_vertices` locations for 20 iterations. Update the occupancy grid every iteration using `density_fn(x) * render_step_size` [S1, unofficial].
    - **Background Density:** Optimize the background density encoding to match `init_density_value` at `pretrain_bg_vertices` and `pretrain_aug_vertices` locations for 40 iterations [S1, unofficial].
    - Set `grid_initialized = True` [S1, unofficial].

3.  **Training Loop Callbacks:**
    - **Step Increment:** Increment `training_step` after each iteration.
    - **Occupancy Update:** Before each training iteration, update the occupancy grid if `training_step > occ_grid_update_warmup_step`. The update function uses `occ_eval_fn = lambda x: field.density_fn(x) * render_step_size` and threshold `alpha_thre` [S1, unofficial].

4.  **Ray Marching and Integration (Forward Pass):**
    - **Determine Fine Samples:** Calculate `num_fine_samples` based on `training_step`:
        - If `training_step < pdf_samples_warmup_step`: `num_fine_samples = pdf_num_samples_per_ray`.
        - If `pdf_samples_warmup_step <= training_step < pdf_samples_fixed_step`:
          $$ \text{ratio} = \frac{\text{training\_step} - \text{pdf\_samples\_warmup\_step}}{\text{pdf\_samples\_fixed\_step} - \text{pdf\_samples\_warmup\_step}} $$
          $$ \text{max\_ratio} = 1 - \text{pdf\_samples\_fixed\_ratio} $$
          $$ \text{num\_fine\_samples} = \text{round}(\text{pdf\_num\_samples\_per\_ray} \times (1 - \text{max\_ratio} \times \text{ratio})) $$
        - If `training_step >= pdf_samples_fixed_step`:
          $$ \text{num\_fine\_samples} = \text{round}(\text{pdf\_num\_samples\_per\_ray} \times \text{pdf\_samples\_fixed\_ratio}) $$
    - **Sampling:** Sample rays using the sampler with `ray_bundle`, `near_plane`, `far_plane`, `render_step_size`, `alpha_thre`, `cone_angle`, and `num_fine_samples`. Returns `ray_samples` and `ray_indices` [S1, unofficial].
    - **Field Query:** Query the neural field with `ray_samples` to obtain densities ($\sigma$) and colors ($C$).
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
            - If `bg_color` is "random": Sample $C_{bg} \sim \text{Uniform}(0, 1)$ per pixel.
        - Final RGB: $C_{final} = C_{comp} + C_{bg} (1 - A)$ [S1, unofficial].
        - If not training, clamp $C_{final}$ to $[0, 1]$ [S1, unofficial].
    - **Depth Estimation:**
        - Compute step centers: $t_i = \frac{t_{start, i} + t_{end, i}}{2}$.
        - Weighted sum: $D = \frac{\sum w_i t_i}{A + \epsilon}$, where $\epsilon = 10^{-10}$.
        - Clip $D$ to $[t_{min}, t_{max}]$ [S1, unofficial].

5.  **Loss and Metrics:**
    - Compute PSNR between $C_{final}$ and ground truth image.
    - If residual RGB loss is enabled, compute mean residual and add to total loss with coefficient `loss_coefficients["res_rgb_loss"]` [S1, unofficial].

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

    # 2. Sample rays
    # Uses occupancy grid for pruning if enabled
    ray_samples, ray_indices = sampler.sample(
        ray_bundle=ray_bundle,
        near_plane=config.near_plane,
        far_plane=config.far_plane,
        render_step_size=config.render_step_size,
        alpha_thre=config.alpha_thre,
        cone_angle=config.cone_angle,
        num_fine_samples=num_fine_samples
    )

    # 3. Query Neural Field
    # Returns densities (sigmas) and colors (RGB)
    sigmas, colors = field.query(ray_samples)

    # 4. Pack info for efficient accumulation
    packed_info = nerfacc.pack_info(ray_indices, num_rays=len(ray_bundle))

    # 5. Compute weights from density
    # Uses frustum starts/ends from ray_samples
    weights = nerfacc.render_weight_from_density(
        t_starts=ray_samples.frustums.starts[..., 0],
        t_ends=ray_samples.frustums.ends[..., 0],
        sigmas=sigmas[..., 0],
        packed_info=packed_info
    )

    # 6. Accumulate RGB
    comp_rgb = nerfacc.accumulate_along_rays(
        weights=weights,
        values=colors,
        ray_indices=ray_indices,
        n_rays=len(ray_bundle)
    )

    # 7. Accumulate Opacity
    accumulation = nerfacc.accumulate_along_rays(
        weights=weights,
        values=None,
        ray_indices=ray_indices,
        n_rays=len(ray_bundle)
    )

    # 8. Composite with Background
    if config.background_color == "black":
        bg_color = torch.tensor([0.0, 0.0, 0.0], device=colors.device)
    elif config.background_color == "white":
        bg_color = torch.tensor([1.0, 1.0, 1.0], device=colors.device)
    else:
        # Random background
        bg_color = torch.rand_like(comp_rgb)

    rgb = comp_rgb + bg_color * (1.0 - accumulation)

    if not training_step: # Inference mode
        rgb = torch.clamp(rgb, 0.0, 1.0)

    # 9. Compute Depth
    steps = (ray_samples.frustums.starts + ray_samples.frustums.ends) / 2.0
    depth = nerfacc.accumulate_along_rays(
        weights=weights,
        values=steps,
        ray_indices=ray_indices,
        n_rays=len(ray_bundle)
    )
    depth = depth / (accumulation + 1e-10)
    depth = torch.clamp(depth, steps.min(), steps.max())

    # 10. Return Outputs
    outputs = {
        'rgb': rgb,
        'accumulation': accumulation,
        'depth': depth,
        'num_samples_per_ray': ray_samples.num_samples
    }

    # Optional: Residual Loss
    if training_step and hasattr(config, 'res_rgb') and config.res_rgb is not None:
        res_rgb_loss = torch.mean(torch.abs(outputs['rgb'] - config.res_rgb))
        outputs['res_rgb_loss'] = res_rgb_loss

    return outputs
```

## Subtopics

### [NeRF Evaluation and Metric Calculation](3079-nerf-evaluation-and-metric-calculation.md)
Computes PSNR, SSIM, and LPIPS for quality assessment.

### [NeRF Network Query with Hashed Embeddings](3107-nerf-network-query-with-hashed-embeddings.md)
Details the TCNN-based hash encoding used for density and color features.

### [Chunked Network Evaluation](3177-chunked-network-evaluation.md)
Handles memory constraints by evaluating rays in chunks.

### [Differentiable Volume Rendering Pipeline](5583-differentiable-volume-rendering-pipeline.md)
The underlying differentiable integration framework.

### [Active Learning Strategy for NeRF Distillation](7421-active-learning-strategy-for-nerf-distillation.md)
Uses uncertainty to select rays for distillation.

### [Direct Matching Alignment Loss](13275-direct-matching-alignment-loss.md)
Loss function for aligning distilled features.

### [Volume Rendering Integration and Forward Pass](15462-volume-rendering-integration-and-forward-pass.md)
General volume rendering math.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of rays and samples per ray. The hash encoding lookup is approximately $O(1)$ average case.
- **Memory Complexity:** Dominated by the hash grids for density and color. The size is determined by `density_log2_hashmap_size` (24) and `color_log2_hashmap_size` (19), allowing for high-resolution representations in limited memory [S1, unofficial].
- **Performance:** Significantly faster training than standard NeRF due to occupancy grid pruning and adaptive sampling. The hybrid representation reduces the number of MLP queries required for empty space [S1, unofficial].
- **Strengths:** Efficient training on large-scale real-world datasets (KITTI-360, Argoverse2). Good generalization with point cloud initialization.
- **Limitations:** Performance depends heavily on the quality of the input point cloud. The adaptive sampling schedule parameters require tuning for different scenes.
- **When to use:** Large-scale outdoor or indoor scenes where standard NeRF is too slow or memory-intensive.
- **When not to use:** Small-scale synthetic scenes (e.g., Blender) where `cone_angle` should be 0.0 and simpler representations may suffice [S1, unofficial].

## Background
Lightning-NeRF was introduced by the Vision-SJTU group as an efficient neural rendering method for large-scale scenes [S1, unofficial]. It builds upon the principles of Instant-NGP (hash encoding) and NeRF (volume rendering) but introduces specific optimizations for real-world data, including point cloud initialization and background augmentation. The method is implemented in the Lightning-NeRF repository [S1, unofficial].

## Sources
[S1] Lightning-NeRF, https://github.com/VISION-SJTU/Lightning-NeRF, fetched 2026-10-29 (unofficial)
