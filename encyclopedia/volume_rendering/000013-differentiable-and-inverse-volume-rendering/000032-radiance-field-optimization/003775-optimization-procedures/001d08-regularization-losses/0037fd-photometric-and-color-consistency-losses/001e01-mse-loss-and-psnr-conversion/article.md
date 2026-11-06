# MSE Loss and PSNR Conversion

Also called: img2mse, MSE Loss, Mean Squared Error Loss, MSE (Mean Squared Error), Mean Squared Error (MSE)

## Overview
MSE Loss computes the Mean Squared Error between predicted and target RGB values, serving as the primary differentiable loss function for training neural volume rendering models [S1, unofficial][S2, unofficial][S4, unofficial]. It is distinguished from structural metrics like SSIM by measuring pixel-level photometric error averaged over the entire frame, making it sensitive to large smooth regions but less responsive to small, high-frequency details [S1, unofficial]. The method often includes a conversion to Peak Signal-to-Noise Ratio (PSNR) for logging and evaluation, providing a logarithmic measure of fidelity where higher values indicate better reconstruction [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Predicted RGB tensor $\hat{C}$ (shape: $[B, H, W, 3]$ or flattened $[N, 3]$) and Target RGB tensor $C_{gt}$ (same shape). Optional: Binary mask $M$ for volumetric data.
- **Outputs:** Scalar loss value (MSE) and optionally PSNR.

**Data Structures:**
- **Tensors:** Floating-point arrays (typically `float32`) representing color values normalized to $[0, 1]$.
- **Mask:** Boolean array of same spatial dimensions as input, indicating valid voxels/pixels.

**Steps:**
1. **Compute Element-wise Difference:** Calculate the difference between predicted and target values: $\Delta = \hat{C} - C_{gt}$.
2. **Square the Difference:** Compute $\Delta^2$.
3. **Apply Mask (Optional):** If a mask $M$ is provided, retain only elements where $M$ is true. Let $N$ be the count of valid elements. If no mask, $N$ is the total number of elements.
4. **Average:** Compute the mean of the squared differences over the valid elements:
   $$ \text{MSE} = \frac{1}{N} \sum_{i=1}^{N} (\hat{C}_i - C_{gt,i})^2 $$
   [S1, unofficial][S3, unofficial][S5, unofficial]
5. **Compute PSNR (Optional):** Convert MSE to PSNR for logging:
   $$ \text{PSNR} = -10 \cdot \log_{10}(\text{MSE}) $$
   [S1, unofficial]
   Alternatively, using the standard definition derived from signal-to-noise ratio:
   $$ \text{PSNR} = 20 \cdot \log_{10}\left(\frac{1}{\sqrt{\text{MSE}}}\right) $$
   [S5, unofficial]
   Note: If $\text{MSE} = 0$, PSNR is defined as $\infty$ [S5, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def img2mse(predicted_rgb, target_rgb, mask=None):
    """
    Computes Mean Squared Error between predicted and target RGB values.
    
    Args:
        predicted_rgb: Array of shape (..., 3) or (N, 3) with predicted colors.
        target_rgb: Array of same shape as predicted_rgb with ground truth colors.
        mask: Optional boolean array of shape (...,) or (N,) indicating valid elements.
        
    Returns:
        mse: Scalar float representing the mean squared error.
    """
    # 1. Compute element-wise difference
    diff = predicted_rgb - target_rgb
    
    # 2. Square the difference
    sq_diff = diff ** 2
    
    # 3. Apply mask if provided
    if mask is not None:
        # Ensure mask is broadcastable or reshaped to match spatial dims
        # Flatten spatial dimensions for averaging
        sq_diff_flat = sq_diff.reshape(-1, 3)
        mask_flat = mask.reshape(-1)
        
        # Filter valid elements
        # Note: Assuming mask applies to all channels equally
        valid_sq_diff = sq_diff_flat[mask_flat]
        
        if valid_sq_diff.size == 0:
            raise RuntimeError("Mask has no true values (empty region).")
            
        # 4. Average over valid elements
        # Mean over all channels and valid pixels
        mse = np.mean(valid_sq_diff)
    else:
        # Flatten spatial dimensions
        sq_diff_flat = sq_diff.reshape(-1, 3)
        # 4. Average over all elements
        mse = np.mean(sq_diff_flat)
        
    return mse

def mse2psnr(mse):
    """
    Converts MSE to PSNR.
    
    Args:
        mse: Scalar float representing Mean Squared Error.
        
    Returns:
        psnr: Scalar float representing PSNR. Returns infinity if mse is 0.
    """
    if mse == 0:
        return float('inf')
    
    # Formula: PSNR = -10 * log10(MSE)
    # Equivalent to: 20 * log10(1 / sqrt(MSE))
    psnr = -10.0 * math.log10(mse)
    return psnr
```

**Parameters:**
- `mask_threshold_rel`: Relative threshold applied to the normalized mask volume to create the boolean mask. Default value is `0.01` [S5, unofficial].
- `batch_size`: Number of random pixels sampled per iteration [S4, unofficial].
- `lr`: Adam learning rate [S4, unofficial].

**Edge Cases and Pitfalls:**
- **Empty Mask:** If the boolean mask has no true values, raise a `RuntimeError` [S5, unofficial].
- **Zero MSE:** If `masked_mse` is `0.0`, PSNR remains infinity to avoid division by zero in the `log10` calculation [S5, unofficial].
- **Precision:** Use `float32` or `float64` consistently to avoid precision loss in squared differences, especially for small errors [S6, unofficial].

## Subtopics

### [PSNR Metric Calculation](2742-psnr-metric-calculation.md)
Peak Signal-to-Noise Ratio (PSNR) evaluates the quality of reconstructed images in novel view synthesis and volume rendering by comparing rendered output against ground-truth reference images. It quantifies the logarithmic ratio between the maximum possible pixel intensity and the Mean Squared Error (MSE) between the two images, providing a single scalar value where higher indicates better fidelity. Unlike structural metrics such as SSIM, PSNR measures pixel-level photometric error averaged over the entire frame, making it sensitive to large smooth regions but less responsive to small, high-frequency details like specular highlights.

### [SSIM Metric Calculation](3650-ssim-metric-calculation.md)
The Structural Similarity Index (SSIM) measures the perceived distortion between a rendered image and a ground truth image by comparing local patterns of pixel intensities, assuming the human visual system is highly adapted to structured natural scene information. It serves as a standard benchmark for image quality in radiance-field optimization, complementing pixel-wise metrics like MSE and PSNR. Unlike MSE, which treats all errors equally, SSIM accounts for luminance, contrast, and structural correlations, providing a value between -1 and 1 where 1 indicates identical images.

### [Normal-L2 Score Computation](3842-normal-l2-score-computation.md)
The Normal-L2 score evaluates surface geometry by computing the mean squared L2 distance between predicted surface normals and reference normals from a ground-truth mesh. It is computed only on the intersection of predicted and ground-truth masks, distinguishing it from silhouette metrics like iIoU which evaluate mask alignment separately.

### [Minkowski's Metric](4347-minkowski-s-metric.md)
Minkowski's Metric is a full-reference mathematical distance measure that treats images as 2D signals to quantify dissimilarity between a reference image $x$ and a test image $y$. It generalizes the Mean Squared Error (MSE) by allowing the exponent $\gamma$ to vary, thereby controlling the sensitivity to large errors versus small errors. Unlike its sibling [PSNR Metric Calculation](2742-psnr-metric-calculation.md) which is a logarithmic transformation of MSE, or [SSIM Metric Calculation](3650-ssim-metric-calculation.md) which models structural information, Minkowski's Metric operates directly on pixel-wise intensity differences.

### Masked Volume Metrics
A method to exclude empty froxels from evaluation metrics by including only froxels affected by radiance [S7, unofficial]. A froxel is included if any of its RGB channels contain a non-zero value at any point during the test scenario [S7, unofficial]. The ROI is computed separately for each tested algorithm since different methods may produce non-zero contributions in different regions of the froxel grid even in the same test scenario. In practice, this means that when analyzing an algorithm, the union of its ROI and the ROI of the ground truth is used:
$$ ROI_{effective} = ROI_{scenario} \cup ROI_{truth} $$
where $ROI_{scenario}$ is the set of froxels with non-zero RGB channels during the test scenario, and $ROI_{truth}$ is the set of froxels with non-zero values in the ground truth [S7, unofficial]. This ensures that only the relevant froxels are included for each output while excluding completely empty regions [S7, unofficial].

### MSE Bias-Variance Decomposition
Treating the TAA algorithm as a biased estimator to analyze correctness using Mean Squared Error (MSE), decomposed into variance and bias squared [S7, unofficial]. The MSE is defined as:
$$ \text{MSE}[F] = \mathbb{E}[(F - G)^2] = V[F] + \beta[F]^2 $$
where $F$ is the output of the estimator, $G$ is the true value (ground truth), $V[F]$ is the variance of the estimator, and $\beta[F]$ is its bias [S7, unofficial]. Constant reconstruction errors, in the form of ghosting, behave similarly to estimator bias, and temporal instability, in the form of flickering, corresponds to estimator variance [S7, unofficial].

### Training Loss Computation with Huber and Masking
In some implementations, Huber loss is used instead of MSE for robustness, combined with masking [S8, unofficial]. The process involves:
1. Render predicted RGB and depth values by calling `render_rays_train` with worldspace rays, appearance embeddings, background, and total samples [S8, unofficial].
2. Split predicted RGBD tensor into RGB and depth: `pred_rgbs, pred_depths = jnp.array_split(pred_rgbds, [3], axis=-1)` [S8, unofficial].
3. Blend ground truth RGB with background to match rendering conditions: `gt_rgbs = data.blend_rgba_image_array(imgarr=gt_rgba_f32, bg=bg)` [S8, unofficial].
4. Compute per-ray Huber loss for RGB with delta=0.1, masked by `ray_is_valid`, and averaged over valid rays: `loss_rgb = jnp.where(batch_metrics["ray_is_valid"], optax.huber_loss(pred_rgbs, gt_rgbs, delta=0.1).mean(axis=-1), 0.).sum() / batch_metrics["n_valid_rays"]` [S8, unofficial].
5. Aggregate loss components: `loss = jax.tree_util.tree_reduce(lambda x, y: x + y, batch_metrics["loss"])` [S8, unofficial].

### Area Loss Scaling
Scaling the loss of each pixel by the area of that pixel's footprint in the original image to ensure low-resolution pixels have comparable influence to high-resolution pixels in multiscale datasets [S9]. For multiscale datasets, calculate the area scaling factor based on the downsampling factor (e.g., factor of 2 -> area factor 4, factor of 4 -> area factor 16). Multiply the pixel loss by this factor [S9].

### IoU Loss
A binary segmentation loss metric that computes the ratio of overlapping voxels above a threshold to the total union of predicted and target voxels [S10, unofficial]. The formula is:
$$ \text{IoU} = \frac{|\{v \mid \hat{v} > \tau\} \cap \{v \mid t_v > 0\}|}{|\{v \mid \hat{v} > \tau\} \cup \{v \mid t_v > 0\}|} $$
where $\tau$ is the prediction threshold for binarization, default 0.9 [S10, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- **Time:** $O(N)$ where $N$ is the number of pixels or voxels evaluated. The computation is element-wise and highly parallelizable on GPUs.
- **Memory:** $O(N)$ to store the difference and squared difference tensors. If masking is used, additional memory for the mask is required.

**Measured Performance:**
- MSE is computationally inexpensive and fast to compute, making it suitable for frequent evaluation during training iterations [S4, unofficial].
- PSNR calculation is $O(1)$ given the MSE value.

**Scaling:**
- Scales linearly with the number of pixels/voxels.
- For large volumes, masking can significantly reduce the effective $N$, improving efficiency if the mask is sparse.

**Strengths and Limitations:**
- **Strengths:** Simple to implement, differentiable, widely used as a baseline metric, easy to interpret in terms of pixel error magnitude.
- **Limitations:** Does not account for structural similarity or perceptual quality, can be sensitive to outliers, may lead to blurry results in optimization compared to perceptual losses [S1, unofficial].

**When to Use:**
- Standard training loss for neural radiance fields when structural fidelity is not the primary concern.
- Baseline evaluation metric for comparing different rendering methods.

**When Not to Use:**
- When perceptual quality is more important than pixel-wise accuracy.
- When structural details are critical and MSE might penalize small shifts heavily.

## Background
MSE Loss originated as a standard metric in signal processing and image quality assessment. In the context of neural volume rendering, it was popularized by the NeRF paper [S1, unofficial] as the primary loss function for optimizing the radiance field. It has since become a standard component in most differentiable rendering pipelines.

## Sources
[S1] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S2] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S3] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S4] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S5] ValentinKraft/vgs, https://github.com/ValentinKraft/vgs, fetched 2026-10-27 (unofficial)
[S6] hardik0001/ct-reconstruction-gpu, https://github.com/hardik0001/ct-reconstruction-gpu, fetched 2026-10-27 (unofficial)
[S7] Temporal Anti-Aliasing for Real-Time Froxel-Based Volumetric Rendering - Advances in volumetric temporal anti-aliasing for real-time ren dering of participating media, https://openalex.org/W7218067689, fetched 2026-11-01 (unofficial)
[S8] blurgyy/jaxngp, https://github.com/blurgyy/jaxngp, fetched 2026-10-27 (unofficial)
[S9] Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00580, fetched 2026-10-26
[S10] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
