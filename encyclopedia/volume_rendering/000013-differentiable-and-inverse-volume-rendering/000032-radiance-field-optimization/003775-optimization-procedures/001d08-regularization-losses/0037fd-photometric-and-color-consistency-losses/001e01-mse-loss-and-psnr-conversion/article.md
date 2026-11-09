# MSE Loss and PSNR Conversion

Also called: MSE (Mean Squared Error), MSE Calculation, MSE Loss, Mean Squared Error, Mean Squared Error Loss, img2mse

## Overview
MSE Loss and PSNR Conversion computes the Mean Squared Error (MSE) between predicted and target RGB values (or volume voxels) to serve as the primary photometric loss for training differentiable volume rendering models, and converts this error into Peak Signal-to-Noise Ratio (PSNR) for logging and evaluation [S2, S3, unofficial][S4, unofficial]. It sets itself apart from its siblings by providing the fundamental pixel-wise intensity error metric, whereas SSIM models structural similarity [S2, unofficial][S3, unofficial], Normal-L2 evaluates geometric normals [S1, unofficial], and IoU operates on binarized occupancy [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Predicted RGB tensor $\hat{C}$ (shape $[B, H, W, 3]$ or flattened) and Target RGB tensor $C_{gt}$ (same shape). For masked variants, a binary mask $M$ is also required.
*   **Outputs:** Scalar MSE loss value; PSNR value (for logging).

**Data Structures:**
*   Tensors are typically stored in contiguous memory layouts (e.g., row-major for PyTorch/TensorFlow).
*   Values are usually normalized to $[0, 1]$ or $[0, 255]$ depending on the source, but the formula assumes consistent scaling between $\hat{C}$ and $C_{gt}$ [S2, S3, unofficial].

**Steps:**
1.  **Compute Difference:** Calculate the element-wise difference between predicted and target values: $D = \hat{C} - C_{gt}$.
2.  **Square Difference:** Square the difference element-wise: $D^2 = D \odot D$.
3.  **Average:** Compute the mean of the squared differences over all spatial dimensions and the batch dimension to obtain MSE [S4, unofficial].
    $$
    \text{MSE} = \frac{1}{N} \sum_{i=1}^{N} (\hat{C}_i - C_{gt, i})^2
    $$
    where $N$ is the total number of elements (pixels or voxels) included in the calculation [S2, S3, unofficial].
4.  **Convert to PSNR:** Convert the MSE to PSNR in decibels (dB) using the logarithmic relationship [S2, S7, unofficial]:
    $$
    \text{PSNR} = -10 \cdot \log_{10}(\text{MSE})
    $$
    Alternatively, expressed as:
    $$
    \text{PSNR} = 20 \cdot \log_{10}\left(\frac{1}{\sqrt{\text{MSE}}}\right)
    $$
    [S7, unofficial]. If $\text{MSE} = 0$, PSNR is defined as $\infty$ [S7, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch

def compute_mse_and_psnr(predicted_rgb: torch.Tensor, target_rgb: torch.Tensor) -> tuple:
    """
    Computes MSE and PSNR between predicted and target RGB tensors.
    
    Args:
        predicted_rgb: Tensor of shape (..., 3) or (..., H, W, 3) with values in [0, 1] or [0, 255].
        target_rgb: Tensor of same shape and scale as predicted_rgb.
        
    Returns:
        mse: Scalar float tensor representing Mean Squared Error.
        psnr: Scalar float tensor representing Peak Signal-to-Noise Ratio.
    """
    # 1. Compute element-wise difference
    diff = predicted_rgb - target_rgb
    
    # 2. Square the difference
    squared_diff = diff ** 2
    
    # 3. Compute mean over all dimensions except the channel dimension (if present)
    # Flatten spatial and batch dimensions to compute global mean
    # Assuming last dimension is channels (3)
    if predicted_rgb.dim() > 1:
        # Reshape to (N, 3) where N is batch * height * width
        n_pixels = predicted_rgb.shape[:-1].numel()
        squared_diff_flat = squared_diff.view(-1, 3)
        mse = torch.mean(squared_diff_flat)
    else:
        mse = torch.mean(squared_diff)
        
    # 4. Convert to PSNR
    # Handle zero MSE to avoid log(0)
    if mse == 0:
        psnr = float('inf')
    else:
        psnr = -10.0 * math.log10(mse.item())
        
    return mse, psnr

def compute_masked_mse_and_psnr(predicted_rgb: torch.Tensor, target_rgb: torch.Tensor, mask: torch.Tensor) -> tuple:
    """
    Computes MSE and PSNR only for voxels/pixels where mask is True.
    
    Args:
        predicted_rgb: Tensor of predicted values.
        target_rgb: Tensor of target values.
        mask: Boolean tensor of same spatial shape as predicted_rgb (excluding channel dim).
        
    Returns:
        masked_mse: Scalar float.
        masked_psnr: Scalar float or inf.
    """
    # Apply mask to squared difference
    # Ensure mask broadcasts correctly to include channel dimension if necessary
    diff = predicted_rgb - target_rgb
    squared_diff = diff ** 2
    
    # Expand mask to match tensor shape if needed (e.g., if mask is HxW and tensor is HxWx3)
    if mask.dim() < predicted_rgb.dim():
        mask = mask.unsqueeze(-1)
        
    masked_squared_diff = squared_diff * mask.float()
    
    # Count valid elements
    num_valid = mask.sum().item()
    
    if num_valid == 0:
        raise RuntimeError("Mask has no true values; cannot compute masked MSE.")
        
    # Compute mean over valid elements only
    masked_mse = masked_squared_diff.sum() / num_valid
    
    # Compute PSNR
    if masked_mse == 0:
        masked_psnr = float('inf')
    else:
        masked_psnr = -10.0 * math.log10(masked_mse.item())
        
    return masked_mse, masked_psnr
```

**Parameters:**
*   **Batch Size:** Number of random pixels or rays sampled per iteration [S5, unofficial].
*   **Learning Rate (lr):** Adam optimizer learning rate, typically from configuration (e.g., `cfg.training.lr`) [S5, unofficial].
*   **Mask Threshold (`mask_threshold_rel`):** Relative threshold applied to the normalized mask volume to create the boolean mask for masked MSE. Default value is `0.01` [S7, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero MSE:** If the prediction perfectly matches the target, MSE is 0. The logarithm in PSNR calculation is undefined. The standard convention is to return $\infty$ or a large finite number [S7, unofficial].
*   **Empty Mask:** If a boolean mask used for masked MSE has no `True` values, the operation is undefined. A `RuntimeError` should be raised [S7, unofficial].
*   **Scale Consistency:** MSE is sensitive to the scale of pixel values (e.g., $[0,1]$ vs $[0,255]$). Ensure predicted and target tensors are normalized to the same range before computation [S2, S3, unofficial].
*   **Gradient Flow:** MSE is differentiable everywhere, allowing standard backpropagation through the rendering equation [S3, unofficial].

## Subtopics

### [PSNR Metric Calculation](2742-psnr-metric-calculation.md)
Peak Signal-to-Noise Ratio (PSNR) is a standard benchmark metric used to evaluate the photometric fidelity of reconstructed or rendered images against ground-truth references in differentiable volume rendering pipelines. It quantifies the mean squared error (MSE) between predicted and target pixel values, converting this error into a logarithmic decibel (dB) scale where higher values indicate better quality. Unlike structural metrics such as SSIM, PSNR focuses strictly on pixel-wise intensity differences, making it sensitive to large smooth regions but less responsive to small, high-frequency features like specular highlights. [PSNR Metric Calculation](2742-psnr-metric-calculation.md)

### [SSIM Metric Calculation](3650-ssim-metric-calculation.md)
SSIM is an image fidelity score used to evaluate the quality of 3D reconstructions and novel 2D views by measuring local structural similarity between a rendered image and a ground truth image. Unlike pixel-wise metrics such as MSE, SSIM captures local patterns of pixel intensities and is correlated with human visual system perception, assuming the human visual system is highly adapted to structured natural scene information. It sets itself apart from its siblings by explicitly modeling luminance, contrast, and structure, whereas PSNR relies solely on global error magnitude and Normal-L2 focuses on geometric normals. [SSIM Metric Calculation](3650-ssim-metric-calculation.md)

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

### Masked Volume Metrics
A method to exclude empty froxels from evaluation metrics by including only froxels affected by radiance [S9, unofficial]. A froxel is included if any of its RGB channels contain a non-zero value at any point during the test scenario [S9, unofficial]. The Region of Interest (ROI) is computed separately for each tested algorithm; the union of the scenario's ROI and the ground truth's ROI is used to create an effective ROI, ensuring only relevant froxels are included while excluding completely empty regions [S9, unofficial]. The formula for the effective ROI is $ROI_{effective} = ROI_{scenario} \cup ROI_{truth}$, where $ROI_{scenario}$ is the set of froxels with non-zero RGB channels during the test scenario, and $ROI_{truth}$ is the set of froxels with non-zero values in the ground truth [S9, unofficial].

### MSE Bias-Variance Decomposition
Treating the TAA algorithm as a biased estimator to analyze correctness using Mean Squared Error (MSE), decomposed into variance and bias squared [S9, unofficial]. The MSE is defined as $\text{MSE}[F] = \mathbb{E}[(F - G)^2] = V[F] + \beta[F]^2$, where $F$ is the output of the estimator, $G$ is the true value (ground truth), $V[F]$ is the variance of the estimator, and $\beta[F]$ is its bias [S9, unofficial].

### Area Loss Scaling
Scaling the loss of each pixel by the area of that pixel's footprint in the original image to ensure low-resolution pixels have comparable influence to high-resolution pixels in multiscale datasets [S10]. For multiscale datasets, the area scaling factor is calculated based on the downsampling factor (e.g., factor of 2 -> area factor 4, factor of 4 -> area factor 16), and the pixel loss is multiplied by this factor [S10].

### Greedy proposal matching
A fast assignment strategy used during Phase 1 to compute the L2 matching loss between heuristic samples and learnt proposals [S11, unofficial]. For each heuristic sample, the closest proposal from the learnt proposer set is found [S11, unofficial]. Optimal matching (Hungarian algorithm) or strict 1-1 greedy matching significantly slows training [S11, unofficial].

### Depth Accuracy Metrics
Metrics used to evaluate depth prior quality: ABSREL (Mean Absolute Relative Error) and RMSE (Root Mean Squared Error) [S12].

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of pixels or voxels compared. The operation is element-wise and parallelizable on GPUs [S4, S5, unofficial].
*   **Memory Complexity:** $O(N)$ to store the difference and squared difference tensors, or $O(1)$ if computed incrementally [S5, unofficial].
*   **Measured Performance:** MSE is the standard loss function for training NeRF and its variants, providing stable gradients for optimization [S3, S5, unofficial]. PSNR is widely reported as the primary metric for evaluating reconstruction quality, with higher values indicating better fidelity [S1, S2, S3, unofficial].
*   **Strengths:** Simple to implement, differentiable, and provides a clear scalar metric for convergence [S3, S5, unofficial].
*   **Limitations:** MSE is sensitive to outliers and does not correlate perfectly with human perception, which is why SSIM and other metrics are often used alongside it [S1, S2, unofficial]. It assumes pixel-wise independence, ignoring structural information [S2, S3, unofficial].
*   **When to Use:** As the primary loss function for training differentiable volume rendering models when photometric fidelity is the main goal [S3, S5, unofficial].
*   **When Not to Use:** When structural similarity is more important than pixel-wise accuracy (use SSIM) [S2, S3, unofficial], or when dealing with noisy data where robust losses like Huber are preferred [S11, unofficial].

## Background
MSE and PSNR are standard metrics in image processing and computer vision. In the context of differentiable volume rendering, MSE was popularized by the NeRF paper and subsequent implementations as the primary photometric loss [S2, S3, unofficial]. PSNR is derived from MSE and is used extensively for benchmarking reconstruction quality [S1, S2, S3, unofficial]. The method is foundational to the training of neural radiance fields and related techniques [S5, unofficial].

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
