# LangSplat Training Procedure

Also called: LangSplat Optimization

## Overview
LangSplat is a training method for 3D Gaussian Splatting that incorporates language features into the rendering pipeline, enabling semantic understanding alongside photometric rendering [S1, unofficial]. It differs from standard 3D Gaussian Splatting by optimizing a loss that includes a language feature reconstruction term, and it disables the standard densification and pruning mechanisms during this specific training mode [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**: 3D Gaussian parameters $\Theta_i = \{x_i, q_i, s_i, \alpha_i, c_i, f_i\}$ [G], camera poses and intrinsics [G], pre-computed language feature maps for training images [S1, unofficial], optimization hyperparameters [S1, unofficial].
*   **Outputs**: Optimized 3D Gaussians with photometric and semantic attributes [S1, unofficial].

**Data Structures**
The method extends the standard Gaussian representation [G]. Each Gaussian $i$ stores:
*   Position $x_i \in \mathbb{R}^3$ [G]
*   Rotation $q_i \in \mathbb{R}^4$ (quaternion) [G]
*   Scale $s_i \in \mathbb{R}^3$ [G]
*   Opacity $\alpha_i \in [0, 1]$ [G]
*   Color coefficients $c_i$ (Spherical Harmonics) [G]
*   Language feature $f_i \in \mathbb{R}^d$ (dimension $d=512$ for CLIP) [S1, unofficial]

**Parameters**
*   `opt.include_feature` (bool): Determines whether to train with language feature loss [S1, unofficial].
*   `opt.lambda_dssim` (float): Weight for the SSIM loss component in photometric mode [S1, unofficial].
*   `opt.densify_until_iter` (int): Iteration after which densification stops [S1, unofficial].
*   `opt.densify_from_iter` (int): Iteration after which densification begins [S1, unofficial].
*   `opt.densification_interval` (int): Interval between densification steps [S1, unofficial].
*   `opt.opacity_reset_interval` (int): Interval for resetting opacity [S1, unofficial].
*   `opt.densify_grad_threshold` (float): Gradient threshold for densification [S1, unofficial].
*   `dataset.feature_level` (int): Level of language feature to use [S1, unofficial].
*   `dataset.lf_path` (str): Path to the directory containing pre-computed language features [S1, unofficial].

**Algorithm Steps**
1.  Initialize Gaussians and scene [G]. Load a checkpoint if provided [S1, unofficial]. Set the background color [G]. Initialize CUDA timing events [S1, unofficial].
2.  Loop over iterations $t$:
    a.  Update learning rates for the optimizer [S1, unofficial].
    b.  Increase the Spherical Harmonics (SH) degree every 1000 iterations [S1, unofficial].
    c.  Pick a random camera from the training set [S1, unofficial].
    d.  Render the scene to obtain the image, language feature map, viewspace points, visibility filter, and radii for each Gaussian [S1, unofficial].
    e.  Compute loss based on `opt.include_feature` [S1, unofficial]. If `true`, compute L1 loss between rendered language feature and ground truth language feature [S1, unofficial]. If `false`, compute L1 and SSIM loss against ground truth image [S1, unofficial].
    f.  Backpropagate the loss [S1, unofficial].
    g.  Update exponential moving average (EMA) of the loss [S1, unofficial].
    h.  Log metrics and save checkpoints [S1, unofficial].
    i.  Densify and prune if not including feature [S1, unofficial].
    j.  Step the optimizer [S1, unofficial].
    k.  Save checkpoint [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def l1_loss(pred, target):
    return F.l1_loss(pred, target)

def ssim(pred, target):
    # Standard structural similarity index computation [G]
    return 1.0 - F.mse_loss(pred, target)

def update_learning_rate(optimizer, iteration):
    # Standard learning rate scheduling [G]
    pass

def update_ema_loss(loss):
    # Update exponential moving average of loss [G]
    pass

def log_metrics(iteration, loss):
    # Log training metrics [G]
    pass

def save_checkpoint(gaussians, iteration):
    # Save model state [G]
    pass

def render(viewpoint_cam, gaussians, pipe, background, opt):
    # Standard differentiable splatting rasterizer [G]
    pass

def train_langsplat(gaussians, viewpoint_cam, pipe, background, opt, dataset, iteration, optimizer, scene):
    SAVE_INTERVAL = 1000  # [G]
    update_learning_rate(optimizer, iteration)

    if iteration % 1000 == 0:
        gaussians.oneupSHdegree()

    render_pkg = render(viewpoint_cam, gaussians, pipe, background, opt)
    image = render_pkg['image']
    language_feature = render_pkg['language_feature']
    viewspace_point_tensor = render_pkg['viewspace_points']
    visibility_filter = render_pkg['visibility_filter']
    radii = render_pkg['radii']

    if opt.include_feature:
        gt_language_feature, language_feature_mask = viewpoint_cam.get_language_feature(
            language_feature_dir=dataset.lf_path,
            feature_level=dataset.feature_level
        )
        Ll1 = l1_loss(language_feature * language_feature_mask, gt_language_feature * language_feature_mask)
        loss = Ll1
    else:
        gt_image = viewpoint_cam.original_image.cuda()
        Ll1 = l1_loss(image, gt_image)
        Lssim = 1.0 - ssim(image, gt_image)
        loss = (1.0 - opt.lambda_dssim) * Ll1 + opt.lambda_dssim * Lssim

    loss.backward()
    update_ema_loss(loss)
    log_metrics(iteration, loss)

    if iteration % SAVE_INTERVAL == 0:
        save_checkpoint(gaussians, iteration)

    if not opt.include_feature:
        if iteration < opt.densify_until_iter:
            gaussians.max_radii2D[visibility_filter] = torch.max(
                gaussians.max_radii2D[visibility_filter],
                radii[visibility_filter]
            )
            gaussians.add_densification_stats(viewspace_point_tensor, visibility_filter)

            if iteration > opt.densify_from_iter and iteration % opt.densification_interval == 0:
                size_threshold = 20 if iteration > opt.opacity_reset_interval else None
                gaussians.densify_and_prune(
                    opt.densify_grad_threshold,
                    0.005,
                    scene.cameras_extent,
                    size_threshold
                )

            if iteration % opt.opacity_reset_interval == 0 or (
                dataset.white_background and iteration == opt.densify_from_iter
            ):
                gaussians.reset_opacity()

    optimizer.step()
```

**Edge Cases and Pitfalls**
*   **Densification Disabled**: When `opt.include_feature` is `true`, densification and pruning are explicitly disabled [S1, unofficial].
*   **Checkpoint Loading**: If loading a checkpoint that contains 12 model parameters (indicating language features are present), `opt.include_feature` must be `true` [S1, unofficial]. Otherwise, `first_iter` should be reset to 0 to avoid state mismatch [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Similar to standard 3D Gaussian Splatting per iteration, but with additional overhead for rendering and computing the language feature loss [S1, unofficial]. The absence of densification/pruning during language training may reduce per-iteration cost slightly compared to the early stages of standard training, but the total training time depends on the number of iterations required for convergence [S1, unofficial].
*   **Memory Complexity**: Increased memory usage due to the storage of language feature vectors $f_i$ for each Gaussian, in addition to standard parameters [S1, unofficial]. The pre-computed language features for training images also consume disk and memory [S1, unofficial].
*   **Performance**: Not given in the sources.
*   **Strengths**: Enables semantic segmentation and language-guided editing directly from the 3D representation [S1, unofficial].
*   **Limitations**: Densification is disabled during language training, which may limit the geometric detail achievable in semantic regions [S1, unofficial]. Requires pre-computation of language features for all training images [S1, unofficial].
*   **When to use**: When semantic understanding or language-based interaction is required from the 3D scene [S1, unofficial].
*   **When not to use**: When only photometric reconstruction is needed, or when high geometric detail is critical and cannot be achieved without densification [S1, unofficial].

## Background
LangSplat was introduced by Minghan Qin et al. as an extension of 3D Gaussian Splatting to incorporate language features [S1, unofficial]. It leverages pre-trained vision-language models (like CLIP) to embed semantic information into the Gaussian attributes [S1, unofficial]. The method is part of the broader effort to integrate semantic and language understanding into differentiable rendering pipelines [S1, unofficial].

## Sources
[S1] minghanqin/LangSplat, https://github.com/minghanqin/LangSplat, fetched 2026-10-27 (unofficial)
