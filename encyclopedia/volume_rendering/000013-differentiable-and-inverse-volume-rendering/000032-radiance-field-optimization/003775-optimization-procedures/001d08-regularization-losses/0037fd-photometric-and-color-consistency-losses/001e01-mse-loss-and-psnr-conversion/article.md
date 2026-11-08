# MSE Loss and PSNR Conversion

Also called: MSE (Mean Squared Error), MSE Calculation, MSE Loss, Mean Squared Error, Mean Squared Error Loss, img2mse

## Overview
MSE Loss and PSNR Conversion computes the Mean Squared Error (MSE) between predicted and target RGB values to serve as the primary photometric loss function for training differentiable volume rendering models, and converts this error into Peak Signal-to-Noise Ratio (PSNR) for logging and evaluation [S2, S3, S5, unofficial]. Unlike structural metrics such as SSIM, MSE focuses strictly on pixel-wise intensity differences, making it sensitive to large smooth regions but less responsive to small, high-frequency features like specular highlights [S1, unofficial]. It builds on [Photometric and Color Consistency Losses](8132-photometric-and-color-consistency-losses.md) by providing the fundamental differentiable objective for minimizing radiance field reconstruction error.

## How it works
**Inputs and Outputs:**
- **Inputs:** Predicted RGB tensor $\hat{C}$ (shape: $[B, H, W, 3]$ or flattened $[N, 3]$) and Target RGB tensor $C_{gt}$ (same shape and dtype, typically normalized to $[0, 1]$ or $[0, 255]$).
- **Outputs:** Scalar MSE loss value and PSNR value (in dB).

**Data Structures:**
- Tensors are stored in standard floating-point format (FP32 or FP16).
- Spatial dimensions are flattened to a single dimension of size $N = B \times H \times W$ for efficient vectorized reduction [S4, unofficial].

**Steps:**
1. **Compute Element-wise Difference:** Calculate the difference between predicted and target values: $D = \hat{C} - C_{gt}$ [S4, unofficial].
2. **Square the Difference:** Compute the squared error for each element: $E = D^2$ [S4, unofficial].
3. **Flatten and Average:** Reshape $E$ to shape $[B, -1]$ to flatten spatial and channel dimensions, then compute the mean along the flattened dimension to obtain the MSE [S4, unofficial].
   $$
   \text{MSE} = \frac{1}{N} \sum_{i=1}^{N} (\hat{C}_i - C_{gt,i})^2
   $$
   where $N$ is the total number of pixels (or voxels in masked variants) [S2, S3, S7, unofficial].
4. **Compute PSNR:** Convert MSE to PSNR using the logarithmic decibel scale [S2, S7, unofficial].
   $$
   \text{PSNR} = -10 \cdot \log_{10}(\text{MSE})
   $$
   Alternatively, if the maximum possible pixel value $M$ is not 1 (e.g., 255 for 8-bit images), the formula is:
   $$
   \text{PSNR} = 20 \cdot \log_{10}\left(\frac{M}{\sqrt{\text{MSE}}}\right)
   $$
   For normalized $[0, 1]$ data, $M=1$, reducing to the first formula [S2, S7, unofficial].

**Parameters:**
- **Batch Size:** Number of random pixels sampled per iteration [S5, unofficial].
- **Learning Rate:** Adam optimizer learning rate (typical values from configuration, e.g., $10^{-3}$ to $10^{-4}$) [S5, unofficial].

**Edge Cases and Pitfalls:**
- **Zero MSE:** If the masked MSE is exactly 0.0, PSNR remains infinity to avoid division by zero in the $\log_{10}$ calculation [S7, unofficial].
- **Empty Mask:** If a boolean mask used for evaluation has no true values (empty region), a `RuntimeError` should be raised [S7, unofficial].
- **Dimension Mismatch:** Input vectors must have the same size; otherwise, an error is thrown [S8, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch

def img2mse(predicted_rgb, target_rgb):
    """
    Computes Mean Squared Error between predicted and target RGB values.
    
    Args:
        predicted_rgb: Tensor of shape (..., 3) or (N, 3)
        target_rgb: Tensor of shape (..., 3) or (N, 3)
        
    Returns:
        mse: Scalar tensor representing the mean squared error
    """
    # Step 1: Compute element-wise difference
    diff = predicted_rgb - target_rgb
    
    # Step 2: Square the difference
    squared_diff = diff ** 2
    
    # Step 3: Flatten spatial dimensions and compute mean
    # Reshape to (batch_size, -1) to flatten all spatial and channel dims
    reshaped = squared_diff.reshape(squared_diff.shape[0], -1)
    
    # Compute mean along dimension 1 (flattened spatial)
    mse = torch.mean(reshaped, dim=1)
    
    # If batch dimension is 1, return scalar; otherwise return mean over batch
    if mse.dim() == 0:
        return mse
    else:
        return torch.mean(mse)

def mse2psnr(mse):
    """
    Converts MSE to Peak Signal-to-Noise Ratio (PSNR).
    
    Args:
        mse: Scalar tensor or float representing Mean Squared Error
        
    Returns:
        psnr: Scalar tensor or float representing PSNR in dB
    """
    # Handle zero MSE to avoid log(0)
    if mse == 0:
        return float('inf')
    
    # Step 4: Compute PSNR using log10
    # PSNR = -10 * log10(MSE) for normalized [0,1] data
    psnr = -10.0 * (math.log10(mse) if isinstance(mse, float) else torch.log10(mse))
    
    return psnr

def compute_masked_metrics(predicted_volume, target_volume, mask_bool):
    """
    Computes masked MSE and PSNR for volumetric data.
    
    Args:
        predicted_volume: Tensor of shape (V, 3) or (B, V, 3)
        target_volume: Tensor of shape (V, 3) or (B, V, 3)
        mask_bool: Boolean tensor of shape (V,) or (B, V,) indicating valid voxels
        
    Returns:
        dict: Dictionary containing masked_mse, masked_psnr, masked_voxel_count
    """
    # Apply mask
    masked_pred = predicted_volume[mask_bool]
    masked_target = target_volume[mask_bool]
    
    # Check for empty mask
    if masked_pred.numel() == 0:
        raise RuntimeError("Mask has no true values (empty region)")
    
    # Compute MSE on masked voxels
    mse = img2mse(masked_pred, masked_target)
    
    # Compute PSNR
    psnr = mse2psnr(mse)
    
    # Count valid voxels
    voxel_count = mask_bool.sum().item()
    
    return {
        'masked_mse': mse.item(),
        'masked_psnr': psnr,
        'masked_voxel_count': voxel_count
    }
```

## Subtopics

### [PSNR Metric Calculation](2742-psnr-metric-calculation.md)
Peak Signal-to-Noise Ratio (PSNR) is a standard benchmark metric used to evaluate the photometric fidelity of reconstructed or rendered images against ground-truth references in differentiable volume rendering pipelines. It quantifies the mean squared error (MSE) between predicted and target pixel values, converting this error into a logarithmic decibel (dB) scale where higher values indicate better quality. Unlike structural metrics such as SSIM, PSNR focuses strictly on pixel-wise intensity differences, making it sensitive to large smooth regions but less responsive to small, high-frequency features like specular highlights. [PSNR Metric Calculation](2742-psnr-metric-calculation.md)

### [SSIM Metric Calculation](3650-ssim-metric-calculation.md)
The Structural Similarity Index (SSIM) measures the perceived distortion between a rendered image and a ground truth image by comparing local patterns of pixel intensities, assuming the human visual system is highly adapted to structured natural scene information. It serves as a standard benchmark for image quality in radiance-field optimization, complementing pixel-wise metrics like MSE and PSNR. It provides a value between -1 and 1, where 1 indicates identical images. [SSIM Metric Calculation](3650-ssim-metric-calculation.md)

### [Normal-L2 Score Computation](3842-normal-l2-score-computation.md)
The Normal-L2 score evaluates surface geometry by computing the mean squared L2 distance between predicted surface normals and reference normals from a ground-truth mesh. It is computed only on the intersection of predicted and ground-truth masks, distinguishing it from silhouette metrics like iIoU which evaluate mask alignment separately. [Normal-L2 Score Computation](3842-normal-l2-score-computation.md)

### [Minkowski's Metric](4347-minkowski-s-metric.md)
Minkowski's metric is a family of distance measures that treats an image as a 2D signal to calculate the dissimilarity between a reference image and a test image. It generalizes the Mean Squared Error (MSE) by allowing the order parameter $\gamma$ to vary, enabling different sensitivities to pixel-wise errors. Unlike its sibling [PSNR](2742-psnr-metric-calculation.md), which normalizes error by signal power to provide a logarithmic decibel scale, Minkowski's metric provides a raw, unnormalized magnitude of error. [Minkowski's Metric](4347-minkowski-s-metric.md)

### [Training Loss Computation with Huber and Masking](4765-training-loss-computation-with-huber-and-masking.md)
This method computes the training loss for differentiable volume rendering by combining a Huber loss on RGB values with optional masking terms. It addresses the sensitivity of Mean Squared Error (MSE) to outliers in photometric consistency by using a robust loss function, while applying masks to exclude invalid rays or background regions. It builds on [MSE Loss and PSNR Conversion](1651-mse-loss-and-psnr-conversion.md) by replacing the standard MSE with a Huber variant and introducing ray validity masking. [Training Loss Computation with Huber and Masking](4765-training-loss-computation-with-huber-and-masking.md)

### [IoU Loss](7110-iou-loss.md)
IoU Loss is a binary segmentation metric used to optimize the occupancy or density fields in differentiable volume rendering by measuring the overlap between predicted and target voxel sets. It differs from pixel-wise photometric losses like [MSE Loss and PSNR Conversion](1651-mse-loss-and-psnr-conversion.md) by operating on binarized volumetric occupancy rather than continuous RGB values, making it robust to intensity ambiguities and suitable for structural consistency. [IoU Loss](7110-iou-loss.md)

### [Perceptually-motivated variance metric](9969-perceptually-motivated-variance-metric.md)
The perceptually-motivated variance metric quantifies the error between an 8-bit ambient occlusion (AO) approximation $T$ and a high-fidelity ray-traced reference $R$. It differs from raw pixel Mean Squared Error (MSE) by operating on log-transformed values and spatial gradients, thereby aligning with human visual sensitivity to radiance and its changes. This metric is specifically designed for evaluating AO approximations where reflected ambient radiance is linear in accessibility. [Perceptually-motivated variance metric](9969-perceptually-motivated-variance-metric.md)

#### Masked Volume Metrics
A method to exclude empty froxels from evaluation metrics by including only froxels affected by radiance. A froxel is included if any of its RGB channels contain a non-zero value at any point during the test scenario. The union of the scenario's region of interest ($ROI_{scenario}$) and the ground truth's region of interest ($ROI_{truth}$) is computed to create $ROI_{effective} = ROI_{scenario} \cup ROI_{truth}$, ensuring only relevant froxels are included in the analysis. [Masked Volume Metrics](4347-minkowski-s-metric.md) (Note: No dedicated article exists for this variant; it is described here based on source).

#### MSE Bias-Variance Decomposition
Treating the TAA algorithm as a biased estimator to analyze correctness using Mean Squared Error (MSE), decomposed into variance and bias squared. The formula is $\text{MSE}[F] = \mathbb{E}[(F - G)^2] = V[F] + \beta[F]^2$, where $F$ is the output of the estimator, $G$ is the true value (ground truth), $V[F]$ is the variance of the estimator, and $\beta[F]$ is its bias. [MSE Bias-Variance Decomposition](4347-minkowski-s-metric.md) (Note: No dedicated article exists for this variant; it is described here based on source).

#### Area Loss Scaling
Scaling the loss of each pixel by the area of that pixel's footprint in the original image to ensure low-resolution pixels have comparable influence to high-resolution pixels in multiscale datasets. For multiscale datasets, the area scaling factor is calculated based on the downsampling factor (e.g., factor of 2 -> area factor 4, factor of 4 -> area factor 16), and the pixel loss is multiplied by this factor. [Area Loss Scaling](4347-minkowski-s-metric.md) (Note: No dedicated article exists for this variant; it is described here based on source).

#### Greedy proposal matching
A fast assignment strategy used during Phase 1 to compute the L2 matching loss between heuristic samples and learnt proposals. For each heuristic sample, the closest proposal from the learnt proposer set is found. Optimal matching (Hungarian algorithm) or strict 1-1 greedy matching significantly slows training. [Greedy proposal matching](4347-minkowski-s-metric.md) (Note: No dedicated article exists for this variant; it is described here based on source).

#### Depth Accuracy Metrics
Metrics used to evaluate depth prior quality: ABSREL (Mean Absolute Relative Error) and RMSE (Root Mean Squared Error). [Depth Accuracy Metrics](4347-minkowski-s-metric.md) (Note: No dedicated article exists for this variant; it is described here based on source).

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of pixels or voxels, due to element-wise operations and a single reduction pass [S4, unofficial].
- **Memory Complexity:** $O(N)$ for storing intermediate squared differences if not computed in-place, or $O(1)$ if using streaming reduction [S4, unofficial].
- **Measured Performance:** Computation is extremely fast and typically negligible compared to the forward and backward passes of the volume rendering operator [S5, unofficial].
- **Scaling:** Scales linearly with resolution and batch size [S4, unofficial].
- **Strengths:** Simple to implement, differentiable everywhere, and widely used as a baseline for photometric fidelity [S3, unofficial].
- **Limitations:** Sensitive to outliers and does not account for structural similarity or perceptual quality [S1, unofficial].
- **When to use:** Standard training objective for radiance fields when photometric accuracy is the primary goal [S3, unofficial].
- **When not to use:** When structural fidelity or perceptual quality is more important than pixel-wise accuracy, or when dealing with significant outliers [S1, unofficial].

## Background
MSE Loss and PSNR Conversion originated with the introduction of Neural Radiance Fields (NeRF) by Mildenhall et al. in 2020 [S2, unofficial]. It has since become the standard photometric loss function for training differentiable volume rendering models [S3, S5, unofficial]. It is used in a wide range of applications, including 3D reconstruction, novel view synthesis, and inverse rendering [S1, S5, unofficial].

## Sources
[S1] Differentiable Voxel-based X-ray Rendering Improves Sparse-View 3D CBCT Reconstruction, https://doi.org/10.48550/arxiv.2411.19224, fetched 2026-10-26 (unofficial)
[S2] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S3] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S4] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S5] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S6] imanfs/inverse-problems-CT-scans - GitHub, https://github.com/imanfs/inverse-problems-CT-scans, fetched 2026-10-27 (unofficial)
[S7] ValentinKraft/vgs, https://github.com/ValentinKraft/vgs, fetched 2026-10-27 (unofficial)
[S8] hardik0001/ct-reconstruction-gpu, https://github.com/hardik0001/ct-reconstruction-gpu, fetched 2026-10-27 (unofficial)
[S9] Temporal Anti-Aliasing for Real-Time Froxel-Based Volumetric Rendering - Advances in volumetric temporal anti-aliasing for real-time ren dering of participating media, https://openalex.org/W7218067689, fetched 2026-11-01 (unofficial)
[S10] Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00580, fetched 2026-10-26
[S11] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
[S12] Digging into Depth Priors for Outdoor Neural Radiance Fields, https://doi.org/10.1145/3581783.3612306, fetched 2026-10-30
