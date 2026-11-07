# MSE Loss and PSNR Conversion

Also called: MSE (Mean Squared Error), MSE Calculation, MSE Loss, Mean Squared Error, Mean Squared Error Loss, img2mse

## Overview
The Mean Squared Error (MSE) loss computes the average of the squared differences between predicted and target RGB values, serving as the primary photometric consistency objective in differentiable volume rendering [S1, S2, S3, S4, unofficial]. It is often accompanied by the Peak Signal-to-Noise Ratio (PSNR) metric, which converts the MSE into a logarithmic decibel scale to quantify reconstruction fidelity [S1, S2, S3, unofficial]. Unlike structural metrics such as SSIM, MSE focuses strictly on pixel-wise intensity differences, making it sensitive to large smooth regions but less responsive to small, high-frequency features like specular highlights [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Predicted RGB values $\hat{C}$ (shape: $[B, H, W, 3]$ or flattened $[N, 3]$) and Target RGB values $C_{gt}$ (same shape). Optionally, a binary mask $M$ indicating valid voxels or pixels [S6, unofficial].
- **Outputs:** Scalar MSE loss value and corresponding PSNR value (for logging) [S1, S6, unofficial].

**Data Structures:**
- **Predicted/Target Tensors:** Floating-point arrays (typically `float32`) representing color channels. Values are usually normalized to $[0, 1]$ [S2, S4, unofficial].
- **Mask (Optional):** Boolean array of same spatial dimensions as input, where `True` indicates valid regions [S6, unofficial].

**Steps:**
1. **Compute Element-wise Difference:** Calculate the difference between predicted and target values: $D = \hat{C} - C_{gt}$ [S3, unofficial].
2. **Square the Difference:** Compute the squared error: $E = D^2$ [S3, unofficial].
3. **Apply Mask (Optional):** If a mask $M$ is provided, retain only errors where $M$ is true. Let $N$ be the number of valid elements (pixels or voxels) [S6, unofficial].
4. **Average:** Compute the mean of the squared errors over the batch and spatial dimensions:
   $$
   \text{MSE} = \frac{1}{N} \sum_{i=1}^{N} (\hat{C}_i - C_{gt, i})^2
   $$
   where $N$ is the total number of pixels (or voxels in masked case) [S1, S6, unofficial].
5. **Compute PSNR:** Convert MSE to PSNR using the logarithmic formula:
   $$
   \text{PSNR} = -10 \cdot \log_{10}(\text{MSE})
   $$
   Alternatively expressed as:
   $$
   \text{PSNR} = 20 \cdot \log_{10}\left(\frac{1}{\sqrt{\text{MSE}}}\right)
   $$
   [S1, S6, unofficial]. If $\text{MSE} = 0$, PSNR is defined as $\infty$ [S6, unofficial].

**Parameters:**
- **Batch Size:** Number of random pixels sampled per iteration [S4, unofficial].
- **Mask Threshold Relative (`mask_threshold_rel`):** Relative threshold applied to the normalized mask volume to create the boolean mask. Default value is `0.01` [S6, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def compute_mse_and_psnr(predicted: np.ndarray, target: np.ndarray, mask: np.ndarray = None) -> dict:
    """
    Computes MSE and PSNR between predicted and target RGB values.
    
    Args:
        predicted: Array of shape (..., 3) or (N, 3) with predicted RGB values.
        target: Array of same shape as predicted with ground truth RGB values.
        mask: Optional boolean array of spatial dimensions (..., ) indicating valid regions.
        
    Returns:
        Dictionary with 'mse' (float), 'psnr' (float or inf), and 'valid_count' (int).
    """
    # Step 1 & 2: Compute squared difference
    diff = predicted - target
    squared_diff = np.square(diff)
    
    # Step 3: Apply mask if provided
    if mask is not None:
        # Ensure mask broadcasts correctly to the last dimension (RGB)
        # Assuming mask is (H, W) and data is (H, W, 3)
        if mask.ndim == squared_diff.ndim - 1:
            mask_expanded = np.expand_dims(mask, axis=-1)
            mask_expanded = np.broadcast_to(mask_expanded, squared_diff.shape)
        else:
            mask_expanded = mask
            
        masked_squared_diff = squared_diff * mask_expanded.astype(float)
        valid_count = int(np.sum(mask))
        
        if valid_count == 0:
            raise RuntimeError("Mask has no true values (empty region).")
            
        mse = float(np.sum(masked_squared_diff) / valid_count)
    else:
        # Flatten spatial dimensions and average over batch
        # Shape (..., 3) -> (N, 3)
        flat = squared_diff.reshape(-1, squared_diff.shape[-1])
        mse = float(np.mean(flat))
        valid_count = squared_diff.size // squared_diff.shape[-1]

    # Step 5: Compute PSNR
    if mse > 0.0:
        psnr = 20.0 * math.log10(1.0 / math.sqrt(mse))
    else:
        psnr = math.inf

    return {
        'mse': mse,
        'psnr': psnr,
        'valid_count': valid_count
    }
```

**Edge Cases and Pitfalls:**
- **Division by Zero:** If the masked region is empty (no `True` values), a `RuntimeError` should be raised [S6, unofficial].
- **Zero MSE:** If $\text{MSE} = 0$, PSNR is infinite. This should be handled explicitly to avoid `log10(0)` errors [S6, unofficial].
- **Precision:** Use floating-point arithmetic (e.g., `float32` or `float64`) to prevent overflow when squaring large differences, though typical normalized RGB values minimize this risk [S3, S7, unofficial].

## Subtopics

### [PSNR Metric Calculation](2742-psnr-metric-calculation.md)
Peak Signal-to-Noise Ratio (PSNR) is a standard benchmark metric used to evaluate the photometric fidelity of reconstructed or rendered images against ground-truth references in differentiable volume rendering pipelines. It quantifies the mean squared error (MSE) between predicted and target pixel values, converting this error into a logarithmic decibel (dB) scale where higher values indicate better quality. Unlike structural metrics such as SSIM, PSNR focuses strictly on pixel-wise intensity differences, making it sensitive to large smooth regions but less responsive to small, high-frequency features like specular highlights.

### [SSIM Metric Calculation](3650-ssim-metric-calculation.md)
The Structural Similarity Index (SSIM) measures the perceived distortion between a rendered image and a ground truth image by comparing local patterns of pixel intensities, assuming the human visual system is highly adapted to structured natural scene information. It serves as a standard benchmark for image quality in radiance-field optimization, complementing pixel-wise metrics like MSE and PSNR. It provides a value between -1 and 1, where 1 indicates identical images.

### [Normal-L2 Score Computation](3842-normal-l2-score-computation.md)
The Normal-L2 score evaluates surface geometry by computing the mean squared L2 distance between predicted surface normals and reference normals from a ground-truth mesh. It is computed only on the intersection of predicted and ground-truth masks, distinguishing it from silhouette metrics like iIoU which evaluate mask alignment separately.

### [Minkowski's Metric](4347-minkowski-s-metric.md)
Minkowski's Metric is a full-reference mathematical distance measure that treats images as 2D signals to quantify dissimilarity between a reference image $x$ and a test image $y$. It generalizes the Mean Squared Error (MSE) by allowing the exponent $\gamma$ to vary, thereby controlling the sensitivity to large errors versus small errors. Unlike its sibling [PSNR Metric Calculation](2742-psnr-metric-calculation.md) which is a logarithmic transformation of MSE, or [SSIM Metric Calculation](3650-ssim-metric-calculation.md) which models structural information, Minkowski's Metric operates directly on pixel-wise intensity differences.

### [Training Loss Computation with Huber and Masking](4765-training-loss-computation-with-huber-and-masking.md)
This method computes the training loss for differentiable volume rendering by combining a Huber loss on RGB values with optional masking terms. It addresses the sensitivity of Mean Squared Error (MSE) to outliers in photometric consistency by using a robust loss function, while applying masks to exclude invalid rays or background regions. It builds on [MSE Loss and PSNR Conversion](1651-mse-loss-and-psnr-conversion.md) by replacing the standard MSE with a Huber variant and introducing ray validity masking.

#### Masked Volume Metrics
A method to exclude empty froxels from evaluation metrics by including only froxels affected by radiance [S8, unofficial]. A froxel is included if any of its RGB channels contain a non-zero value at any point during the test scenario [S8, unofficial]. The Region of Interest (ROI) is computed separately for each tested algorithm; the union of the scenario's ROI and the ground truth's ROI is used to create $ROI_{effective}$, ensuring only relevant froxels are included while excluding completely empty regions [S8, unofficial].
$$
ROI_{effective} = ROI_{scenario} \cup ROI_{truth}
$$
where $ROI_{scenario}$ is the set of froxels with non-zero RGB channels during the test scenario, and $ROI_{truth}$ is the set of froxels with non-zero values in the ground truth [S8, unofficial].

#### MSE Bias-Variance Decomposition
Treating the TAA algorithm as a biased estimator to analyze correctness using Mean Squared Error (MSE), decomposed into variance and bias squared [S8, unofficial]. This allows separating constant reconstruction errors (ghosting, analogous to bias) from temporal instability (flickering, analogous to variance) [S8, unofficial].
$$
\text{MSE}[F] = \mathbb{E}[(F - G)^2] = V[F] + \beta[F]^2
$$
where $F$ is the output of the estimator, $G$ is the true value (ground truth), $V[F]$ is the variance of the estimator, and $\beta[F]$ is its bias [S8, unofficial].

#### Area Loss Scaling
Scaling the loss of each pixel by the area of that pixel's footprint in the original image to ensure low-resolution pixels have comparable influence to high-resolution pixels in multiscale datasets [S9]. For multiscale datasets, the area scaling factor is calculated based on the downsampling factor (e.g., factor of 2 -> area factor 4, factor of 4 -> area factor 16) and multiplied by the pixel loss [S9].

#### IoU Loss
A binary segmentation loss metric that computes the ratio of overlapping voxels above a threshold to the total union of predicted and target voxels [S10, unofficial].
$$
\text{IoU} = \frac{|\{v \mid \hat{v} > \tau\} \cap \{v \mid t_v > 0\}|}{|\{v \mid \hat{v} > \tau\} \cup \{v \mid t_v > 0\}|}
$$
where $\tau$ is the prediction threshold for binarization (default 0.9) [S10, unofficial].

#### Greedy proposal matching
A fast assignment strategy used during Phase 1 to compute the L2 matching loss between heuristic samples and learnt proposals [S11, unofficial]. For each heuristic sample, the closest proposal from the learnt proposer set is found [S11, unofficial]. Optimal matching (Hungarian algorithm) or strict 1-1 greedy matching significantly slows training, so aggressive greedy matching is preferred for speed [S11, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of pixels or voxels processed. The operation is element-wise and parallelizable on GPUs [S3, S7, unofficial].
- **Memory Complexity:** $O(N)$ for storing intermediate squared differences, or $O(1)$ if computed incrementally [S5, unofficial].
- **Measured Performance:** Standard MSE computation is extremely fast, often taking milliseconds for typical image resolutions (e.g., 512x512) on modern GPUs [S4, S7, unofficial].
- **Strengths:** Simple to implement, differentiable, and widely used as a baseline for photometric accuracy [S2, S4, unofficial].
- **Limitations:** Sensitive to outliers and does not account for structural similarity or perceptual quality [S1, S11, unofficial]. May lead to blurry results compared to perceptual losses [S11, unofficial].
- **When to Use:** When pixel-wise accuracy is the primary goal and computational efficiency is critical. Suitable for initial training phases or when ground truth is high-quality and noise-free [S2, S4, unofficial].
- **When Not to Use:** When structural fidelity or perceptual quality is more important than pixel-wise accuracy, or when the scene contains significant noise or outliers [S1, S11, unofficial].

## Background
The Mean Squared Error (MSE) loss is a fundamental metric in image processing and computer vision, widely adopted in differentiable volume rendering pipelines such as NeRF [S1, S2, S3, S4, unofficial]. It was introduced as a standard loss function for training neural radiance fields to minimize the difference between rendered and ground-truth images [S1, S2, S4, unofficial]. The PSNR metric, derived from MSE, has been used as a standard benchmark for evaluating reconstruction quality in these pipelines [S1, S2, S3, S4, unofficial]. Variants like masked MSE and area-scaled MSE have been developed to handle specific challenges in volumetric rendering, such as empty regions and multiscale datasets [S6, unofficial][S9].

## Sources
[S1] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S2] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S3] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S4] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S5] imanfs/inverse-problems-CT-scans - GitHub, https://github.com/imanfs/inverse-problems-CT-scans, fetched 2026-10-27 (unofficial)
[S6] ValentinKraft/vgs, https://github.com/ValentinKraft/vgs, fetched 2026-10-27 (unofficial)
[S7] hardik0001/ct-reconstruction-gpu, https://github.com/hardik0001/ct-reconstruction-gpu, fetched 2026-10-27 (unofficial)
[S8] Temporal Anti-Aliasing for Real-Time Froxel-Based Volumetric Rendering - Advances in volumetric temporal anti-aliasing for real-time ren dering of participating media, https://openalex.org/W7218067689, fetched 2026-11-01 (unofficial)
[S9] Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00580, fetched 2026-10-26
[S10] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
[S11] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
