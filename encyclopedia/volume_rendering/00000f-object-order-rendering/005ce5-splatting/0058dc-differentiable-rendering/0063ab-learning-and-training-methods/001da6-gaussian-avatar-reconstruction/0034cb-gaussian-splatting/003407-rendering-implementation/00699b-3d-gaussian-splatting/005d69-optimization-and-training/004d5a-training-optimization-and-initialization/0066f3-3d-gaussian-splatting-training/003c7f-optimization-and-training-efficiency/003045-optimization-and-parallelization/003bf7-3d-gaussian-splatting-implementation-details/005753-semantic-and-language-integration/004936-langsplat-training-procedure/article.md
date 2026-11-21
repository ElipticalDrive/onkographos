# LangSplat Training Procedure

Also called: LangSplat Optimization

## Overview
LangSplat is a training method for 3D Gaussian Splatting that incorporates language features into the rendering pipeline, enabling semantic understanding alongside photometric rendering [S1, unofficial]. It differs from standard 3D Gaussian Splatting by optimizing a joint loss that includes a language feature reconstruction term, and it disables the standard densification and pruning mechanisms during this specific training mode to preserve semantic consistency [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**: 3D Gaussian parameters $\Theta_i = \{x_i, q_i, s_i, \alpha_i, c_i, f_i\}$, where $f_i$ is the language feature vector; camera poses and intrinsics; pre-computed language feature maps for training images; optimization hyperparameters.
*   **Outputs**: Optimized 3D Gaussians with photometric and semantic attributes.

**Data Structures**
The method extends the standard Gaussian representation. Each Gaussian $i$ stores:
*   Position $x_i \in \mathbb{R}^3$
*   Rotation $q_i \in \mathbb{R}^4$ (quaternion)
*   Scale $s_i \in \mathbb{R}^3$
*   Opacity $\alpha_i \in [0, 1]$
*   Color coefficients $c_i$ (Spherical Harmonics)
*   Language feature $f_i \in \mathbb{R}^d$ (dimension $d$ depends on the CLIP model, typically 512)

**Parameters**
*   `opt.include_feature` (bool): Determines whether to train with language feature loss. If `true`, only language loss is computed; if `false`, standard photometric loss is used [S1, unofficial].
*   `opt.lambda_dssim` (float): Weight for the SSIM loss component in photometric mode [S1, unofficial].
*   `opt.densify_until_iter` (int): Iteration after which densification stops [S1, unofficial].
*   `opt.densify_from_iter` (int): Iteration after which densification begins [S1, unofficial].
*   `opt.densification_interval` (int): Interval between densification steps [S1, unofficial].
*   `opt.opacity_reset_interval` (int): Interval for resetting opacity [S1, unofficial].
*   `opt.densify_grad_threshold` (float): Gradient threshold for densification [S1, unofficial].
*   `dataset.feature_level` (int): Level of language feature to use [S1, unofficial].
*   `dataset.lf_path` (str): Path to the directory containing pre-computed language features [S1, unofficial].

**Algorithm Steps**
1.  **Initialization**: Initialize 3D Gaussians from the point cloud. Load a checkpoint if provided. Set the background color. Initialize CUDA timing events.
2.  **Iteration Loop**: For each iteration $t$:
    a.  Update learning rates for the optimizer.
    b.  Increase the Spherical Harmonics (SH) degree every 1000 iterations.
    c.  Pick a random camera from the training set.
    d.  **Render**: Render the scene to obtain the image, language feature map, viewspace points, visibility filter, and radii for each Gaussian.
    e.  **Loss Computation**:
        *   If `opt.include_feature` is `true`:
            1.  Retrieve the ground truth language feature $F_{gt}$ and the language feature mask $M$ from the camera's pre-computed data using `dataset.lf_path` and `dataset.feature_level`.
            2.  Compute the L1 loss between the rendered language feature $F_{render}$ and the ground truth, masked by $M$:
                $$ L_{l1} = \| (F_{render} \odot M) - (F_{gt} \odot M) \|_1 $$
            3.  Set total loss $L = L_{l1}$.
        *   If `opt.include_feature` is `false`:
            1.  Retrieve the ground truth image $I_{gt}$.
            2.  Compute L1 loss $L_{l1} = \| I_{render} - I_{gt} \|_1$.
            3.  Compute SSIM loss $L_{ssim} = 1 - \text{SSIM}(I_{render}, I_{gt})$.
            4.  Set total loss $L = (1.0 - \text{opt.lambda\_dssim}) L_{l1} + \text{opt.lambda\_dssim} L_{ssim}$.
    f.  **Backpropagation**: Backpropagate the loss to update Gaussian parameters.
    g.  **Logging**: Update exponential moving average (EMA) of the loss. Log metrics and save checkpoints periodically.
    h.  **Densification and Pruning**:
        *   If `opt.include_feature` is `true`, skip densification and pruning [S1, unofficial].
        *   If `opt.include_feature` is `false` and $t < \text{opt.densify\_until\_iter}$:
            1.  Update the maximum 2D radius for visible Gaussians:
                $$ \max\_radii2D_{visible} = \max(\max\_radii2D_{visible}, radii_{visible}) $$
            2.  Accumulate densification statistics using viewspace points and the visibility filter.
            3.  If $t > \text{opt.densify\_from\_iter}$ and $t \% \text{opt.densification\_interval} == 0$:
                *   Determine size threshold: $20$ if $t > \text{opt.opacity\_reset\_interval}$, else `None`.
                *   Call `densify_and_prune` with `opt.densify_grad_threshold`, opacity threshold $0.005$, scene extent, and size threshold.
            4.  If $t \% \text{opt.opacity\_reset\_interval} == 0$ or (white background and $t == \text{opt.densify\_from\_iter}$), reset opacities.
    i.  **Optimization Step**: Step the optimizer.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_langsplat(gaussians, viewpoint_cam, pipe, background, opt, dataset, iteration):
    # Update learning rate
    update_learning_rate(optimizer, iteration)
    
    # Increase SH degree every 1000 iterations
    if iteration % 1000 == 0:
        gaussians.oneupSHdegree()
    
    # Render scene
    render_pkg = render(viewpoint_cam, gaussians, pipe, background, opt)
    image, language_feature, viewspace_point_tensor, visibility_filter, radii = (
        render_pkg['image'],
        render_pkg['language_feature'],
        render_pkg['viewspace_points'],
        render_pkg['visibility_filter'],
        render_pkg['radii']
    )
    
    # Compute loss
    if opt.include_feature:
        # Get ground truth language feature and mask
        gt_language_feature, language_feature_mask = viewpoint_cam.get_language_feature(
            language_feature_dir=dataset.lf_path,
            feature_level=dataset.feature_level
        )
        # Compute L1 loss with mask
        Ll1 = l1_loss(
            language_feature * language_feature_mask,
            gt_language_feature * language_feature_mask
        )
        loss = Ll1
    else:
        # Get ground truth image
        gt_image = viewpoint_cam.original_image.cuda()
        # Compute L1 and SSIM loss
        Ll1 = l1_loss(image, gt_image)
        Lssim = 1.0 - ssim(image, gt_image)
        loss = (1.0 - opt.lambda_dssim) * Ll1 + opt.lambda_dssim * Lssim
    
    # Backpropagate
    loss.backward()
    
    # Update EMA loss
    update_ema_loss(loss)
    
    # Log and save
    log_metrics(iteration, loss)
    if iteration % save_interval == 0:
        save_checkpoint(gaussians, iteration)
    
    # Densification and pruning (only if not including feature)
    if not opt.include_feature:
        if iteration < opt.densify_until_iter:
            # Update max radii for visible Gaussians
            gaussians.max_radii2D[visibility_filter] = torch.max(
                gaussians.max_radii2D[visibility_filter],
                radii[visibility_filter]
            )
            # Add densification stats
            gaussians.add_densification_stats(viewspace_point_tensor, visibility_filter)
            
            # Perform densification/pruning at intervals
            if iteration > opt.densify_from_iter and iteration % opt.densification_interval == 0:
                size_threshold = 20 if iteration > opt.opacity_reset_interval else None
                gaussians.densify_and_prune(
                    opt.densify_grad_threshold,
                    0.005,
                    scene.cameras_extent,
                    size_threshold
                )
            
            # Reset opacity
            if iteration % opt.opacity_reset_interval == 0 or (
                dataset.white_background and iteration == opt.densify_from_iter
            ):
                gaussians.reset_opacity()
    
    # Step optimizer
    optimizer.step()
```

**Edge Cases and Pitfalls**
*   **Densification Disabled**: When `opt.include_feature` is `true`, densification and pruning are explicitly disabled. This prevents the modification of the Gaussian set, which could disrupt the alignment of language features [S1, unofficial].
*   **Checkpoint Loading**: If loading a checkpoint that contains 12 model parameters (indicating language features are present), `opt.include_feature` must be `true`. Otherwise, `first_iter` should be reset to 0 to avoid state mismatch [S1, unofficial].
*   **Feature Masking**: The language feature loss requires a mask to ignore pixels where no language feature is available (e.g., background or out-of-bounds). Failure to apply this mask correctly will lead to incorrect gradients [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Similar to standard 3D Gaussian Splatting per iteration, but with additional overhead for rendering and computing the language feature loss. The absence of densification/pruning during language training may reduce per-iteration cost slightly compared to the early stages of standard training, but the total training time depends on the number of iterations required for convergence.
*   **Memory Complexity**: Increased memory usage due to the storage of language feature vectors $f_i$ for each Gaussian, in addition to standard parameters. The pre-computed language features for training images also consume disk and memory.
*   **Performance**: Not given in the sources.
*   **Strengths**: Enables semantic segmentation and language-guided editing directly from the 3D representation.
*   **Limitations**: Densification is disabled during language training, which may limit the geometric detail achievable in semantic regions. Requires pre-computation of language features for all training images.
*   **When to use**: When semantic understanding or language-based interaction is required from the 3D scene.
*   **When not to use**: When only photometric reconstruction is needed, or when high geometric detail is critical and cannot be achieved without densification.

## Background
LangSplat was introduced by Minghan Qin et al. as an extension of 3D Gaussian Splatting to incorporate language features. It leverages pre-trained vision-language models (like CLIP) to embed semantic information into the Gaussian attributes. The method is part of the broader effort to integrate semantic and language understanding into differentiable rendering pipelines.

## Sources
[S1] minghanqin/LangSplat, https://github.com/minghanqin/LangSplat, fetched 2026-10-27 (unofficial)
