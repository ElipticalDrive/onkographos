# Training Loss Computation with Huber and Masking

Also called: NeRF training loss, RGB and TV loss calculation

## Overview
This method computes the training loss for differentiable volume rendering by combining a Huber loss on RGB values with optional masking terms. It addresses the sensitivity of Mean Squared Error (MSE) to outliers in photometric consistency by using a robust loss function, while applying masks to exclude invalid rays or background regions. It builds on [MSE Loss and PSNR Conversion](1651-mse-loss-and-psnr-conversion.md) by replacing the standard MSE with a Huber variant and introducing ray validity masking.

## How it works
**Inputs:**
- `pred_rgbds`: Tensor of predicted RGB and depth values from `render_rays_train`. Shape: `(num_rays, 4)` (RGB + Depth).
- `gt_rgba_f32`: Ground truth RGBA image array.
- `bg`: Background color tensor.
- `batch_metrics`: Dictionary containing `ray_is_valid` (boolean mask) and `n_valid_rays` (count).
- `loss_components`: List of additional loss terms (e.g., TV, SH masking) to aggregate.

**Outputs:**
- `loss`: Scalar total loss value.

**Data Structures:**
- `pred_rgbs`: Tensor of shape `(num_rays, 3)` containing predicted RGB.
- `gt_rgbs`: Tensor of shape `(num_rays, 3)` containing blended ground truth RGB.
- `ray_is_valid`: Boolean tensor of shape `(num_rays,)` indicating valid rays.

**Steps:**
1. **Render:** Call `render_rays_train` with worldspace rays, appearance embeddings, background, and total samples to obtain `pred_rgbds`.
2. **Split:** Separate RGB and depth from `pred_rgbds`.
   $$ \text{pred\_rgbs}, \text{pred\_depths} = \text{split}(\text{pred\_rgbds}, \text{axis}=-1, \text{indices\_or\_splits}=[3]) $$
   [G] The split occurs at index 3, separating the first 3 channels (RGB) from the last channel (Depth).
3. **Blend GT:** Blend ground truth RGBA with the background to match rendering conditions.
   $$ \text{gt\_rgbs} = \text{blend\_rgba}(\text{gt\_rgba\_f32}, \text{bg}) $$
   [G] This ensures the ground truth accounts for the same background compositing applied during rendering.
4. **Compute Huber Loss:** Calculate per-ray Huber loss for RGB, masked by validity.
   $$ L_{\text{rgb}} = \frac{\sum_{i} \mathbb{I}(\text{ray\_is\_valid}_i) \cdot \text{Huber}(\text{pred\_rgbs}_i, \text{gt\_rgbs}_i, \delta=0.1)}{N_{\text{valid}}} $$
   Where $\delta=0.1$ is the Huber delta, and $N_{\text{valid}}$ is the count of valid rays.
   [G] The Huber loss is defined as:
   $$ \text{Huber}(p, g, \delta) = \begin{cases} \frac{1}{2}(p-g)^2 & \text{if } |p-g| \le \delta \\ \delta(|p-g| - \frac{1}{2}\delta) & \text{otherwise} \end{cases} $$
5. **Aggregate:** Sum all loss components from `batch_metrics["loss"]`.
   $$ \text{loss} = \sum_{l \in \text{batch\_metrics["loss"]}} l $$

**Parameters:**
- `delta`: Huber loss threshold. Default: 0.1 [S1, unofficial].
- `ray_is_valid`: Mask for valid rays. [S1, unofficial].

**Edge Cases:**
- If `n_valid_rays` is 0, division by zero occurs. [G] Guard against this by checking `n_valid_rays > 0` before division.
- Degenerate rays (e.g., near camera origin) may produce NaNs in depth. [G] Filter out invalid rays using `ray_is_valid`.

## Subtopics

### SH Masking
A variant that incorporates Spherical Harmonics (SH) masking into the loss. It uses a loss weight $\lambda_{sh}=0.05$ and a threshold $\epsilon_{sh}=0.1$ for SH masks, with a learning rate of 0.05 for the masks [S2, unofficial]. This helps regularize the SH coefficients during optimization.

## Operating characteristics
- **Time Complexity:** Dominated by the rendering step `render_rays_train`. The loss computation is $O(N)$ where $N$ is the number of rays.
- **Memory Complexity:** $O(N)$ for storing `pred_rgbs` and `gt_rgbs`.
- **Strengths:** Robust to outliers due to Huber loss. Masking improves training stability by ignoring invalid rays.
- **Limitations:** Requires careful tuning of $\delta$ and mask thresholds. [S2, unofficial] notes that SH masking parameters need specific values.
- **When to use:** When training NeRFs or Gaussian Splatting models where photometric consistency is key and outliers are present.

## Background
Originated in the context of JAX-based NeRF implementations, specifically in the `jaxngp` repository [S1, unofficial]. It is used in differentiable volume rendering pipelines to optimize radiance fields. The SH masking variant is associated with Trick-GS [S2, unofficial].

## Sources
[S1] blurgyy/jaxngp, https://github.com/blurgyy/jaxngp, fetched 2026-10-27 (unofficial)
[S2] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30 (unofficial)
