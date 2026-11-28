# Training Loss Computation with Huber and Masking

Also called: NeRF training loss, RGB and TV loss calculation

## Overview
This method computes the training loss for differentiable volume rendering by applying a Huber loss to rendered RGB values, masking invalid rays, and blending ground truth images against a background color [S1, unofficial]. It extends the parent [Masked and Weighted Losses](15713-masked-and-weighted-losses.md) by detailing the specific blending and aggregation steps for RGBD predictions, while its sibling [Masked Volume Metrics](1927-masked-volume-metrics.md) targets evaluation rather than optimization.

## How it works
**Inputs:**
- `worldspace_rays`: Ray origins and directions for the current batch [S1, unofficial].
- `appearance_embeddings`: Optional per-ray or per-view appearance codes [S1, unofficial].
- `bg`: Background color tensor for blending [S1, unofficial].
- `gt_rgba_f32`: Ground truth RGBA image array in float32 [S1, unofficial].
- `total_samples`: Number of sampling points per ray [S1, unofficial].

**Outputs:**
- `loss`: Scalar training loss value [S1, unofficial].
- `batch_metrics`: Dictionary containing intermediate loss components and validity counts [S1, unofficial].

**Data Structures:**
- `pred_rgbds`: Tensor of shape `(num_rays, num_samples, 4)` containing `[R, G, B, Depth]` [S1, unofficial].
- `ray_is_valid`: Boolean tensor of shape `(num_rays,)` indicating valid rays [S1, unofficial].
- `n_valid_rays`: Integer count of rays where `ray_is_valid` is True [S1, unofficial].

**Parameters:**
- `delta`: Huber loss threshold. Typical value: `0.1` [S1, unofficial].

**Steps:**
1. **Render:** Call the differentiable renderer with `worldspace_rays`, `appearance_embeddings`, `bg`, and `total_samples` to obtain `pred_rgbds` [S1, unofficial].
2. **Split:** Separate the prediction tensor into RGB and Depth components along the last axis [S1, unofficial].
3. **Blend Ground Truth:** Composite the ground truth RGBA image against the background color to match the renderer's assumptions for transparent regions [S1, unofficial].
4. **Compute Per-Ray Huber Loss:** Calculate the Huber loss between predicted and blended ground truth RGB values, average over samples per ray, apply the validity mask, and divide by the number of valid rays [S1, unofficial].
5. **Aggregate:** Sum all loss components stored in `batch_metrics["loss"]` using a tree reduction [S1, unofficial].

**Formulas:**
Huber loss for a single value:
$$ L_{huber}(p, g) = \begin{cases} \frac{1}{2}(p - g)^2 & \text{if } |p - g| \le \delta \\ \delta(|p - g| - \frac{1}{2}\delta) & \text{otherwise} \end{cases} $$ [G].

Per-ray RGB loss:
$$ L_{rgb} = \frac{1}{N_{valid}} \sum_{i=1}^{N_{rays}} \mathbb{I}(\text{ray\_is\_valid}_i) \cdot \frac{1}{N_{samples}} \sum_{s=1}^{N_{samples}} L_{huber}(P_{i,s}, G_{i,s}) $$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_training_loss(pred_rgbds, gt_rgba_f32, bg, ray_is_valid, batch_metrics, delta=0.1):
    # 1. Split RGB and Depth
    pred_rgbs = pred_rgbds[..., :3]
    pred_depths = pred_rgbds[..., 3:]
    
    # 2. Blend Ground Truth RGB with background
    alpha = gt_rgba_f32[..., 3:4]
    gt_rgbs = gt_rgba_f32[..., :3] * alpha + bg * (1.0 - alpha)
    
    # 3. Compute per-ray Huber loss for RGB
    diff = pred_rgbs - gt_rgbs
    abs_diff = np.abs(diff)
    huber = np.where(abs_diff <= delta, 0.5 * diff**2, delta * (abs_diff - 0.5 * delta))
    per_ray_loss = np.mean(huber, axis=-1)
    
    # Mask by validity and sum
    masked_loss = np.where(ray_is_valid, per_ray_loss, 0.0)
    n_valid_rays = np.sum(ray_is_valid.astype(np.float32))
    
    # Avoid division by zero if no valid rays
    loss_rgb = np.where(n_valid_rays > 0, np.sum(masked_loss) / n_valid_rays, 0.0)
    
    # 4. Aggregate with other losses in batch_metrics
    total_loss = sum(batch_metrics["loss"])
    total_loss = total_loss + loss_rgb
    
    return total_loss, {"loss_rgb": loss_rgb, "n_valid_rays": n_valid_rays}
```

**Edge Cases and Pitfalls:**
- Division by zero: Guard `n_valid_rays` to prevent NaN gradients when no rays are valid [G].
- Precision: Very small alpha values or near-zero valid ray counts can cause unstable gradients [G].

## Subtopics

### [SH Masking](9858-sh-masking.md)
SH Masking (SHM) is a regularization technique for differentiable volume rendering optimization that suppresses the influence of saturated pixels during loss computation. Saturated regions (over- or undersaturation) lack reliable texture cues and act as outliers that bias geometric and photometric recovery. It differs from hard binary masking by using a "leaky" quadratic transition at the saturation boundaries, which allows smoother gradient flow while still reducing bias.

### Convergence Hinge Loss
A hinge loss on the mean mask value to ensure both foreground and background generators are used and cover a minimum fraction of the image [S2]. It prevents the model from collapsing to generating only foreground or only background [S2].

**Formula:**
Foreground convergence loss:
$$ L_{cvg}^{fg} = \lambda_{cvg}^{fg} \max\left(0, \eta_{fg} - \frac{1}{|S|} \sum_{i \in S} \hat{m}_{mask}^{fg}[i]\right) $$
Background convergence loss:
$$ L_{cvg}^{bg} = \lambda_{cvg}^{bg} \max\left(0, \eta_{bg} - \frac{1}{|S|} \sum_{i \in S} (1 - \hat{m}_{mask}^{bg}[i])\right) $$
where $\hat{m}_{mask}$ is the predicted mask value, $|S|$ is the number of pixels, and $\eta$ is the minimum coverage fraction [S2].

**Parameters:**
- Foreground minimum coverage ($\eta_{fg}$): 0.4 for AFHQ and FFHQ; 0.1 for Carla [S2].
- Background minimum coverage ($\eta_{bg}$): 0.1 [S2].
- Convergence loss weight ($\lambda_{cvg}^{fg}$): 0.1 [S2].
- Convergence loss weight ($\lambda_{cvg}^{bg}$): 0.1 [S2].

## Operating characteristics
- Time complexity: Not given in the sources.
- Memory complexity: Not given in the sources.
- Measured performance: Not given in the sources.
- Scaling: Not given in the sources.
- Strengths: Not given in the sources.
- Limitations: Not given in the sources.
- When to use: Not given in the sources.
- When not to use: Not given in the sources.

## Background
The method is implemented in the jaxngp repository [S1, unofficial]. The convergence hinge loss variant appears in the VoxGRAF paper for 3D-aware image synthesis [S2].

## Sources
[S1] blurgyy/jaxngp, https://github.com/blurgyy/jaxngp, fetched 2026-10-27 (unofficial)
[S2] VoxGRAF: Fast 3D-Aware Image Synthesis with Sparse Voxel Grids, https://doi.org/10.52202/068431-2464, fetched 2026-10-26
