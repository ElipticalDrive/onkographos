# MSE Loss and PSNR Conversion

Also called: MSE, MSE Loss, Mean Squared Error, Mean Squared Error Loss

## Overview
The Mean Squared Error (MSE) loss computes the average of the squared differences between predicted and target RGB values, serving as the primary photometric optimization objective in differentiable volume rendering [S3, S6, unofficial]. It is distinct from structural metrics like SSIM [S1, unofficial] and geometric losses like IoU [S1, unofficial] by focusing strictly on pixel-wise intensity fidelity [S1, unofficial]. The method often includes a conversion to Peak Signal-to-Noise Ratio (PSNR) for logging, providing a logarithmic scale of reconstruction quality [S2, S3, unofficial].

## How it works
**Inputs:**
- Predicted RGB image tensor $\hat{C}$ (shape: $[B, H, W, 3]$ or flattened $[B, N, 3]$).
- Target RGB image tensor $C_{gt}$ (shape: $[B, H, W, 3]$ or flattened $[B, N, 3]$).
- Optional binary mask $M$ (shape: $[B, H, W]$ or $[B, N]$) indicating valid voxels/pixels [S8, unofficial].

**Outputs:**
- Scalar loss value $L_{MSE}$.
- Optional PSNR value $L_{PSNR}$ [S2, unofficial].

**Data Structures:**
- Tensors are typically stored in contiguous memory layouts (e.g., row-major for PyTorch/TensorFlow) to allow efficient vectorized operations [S5, S6, unofficial].

**Steps:**
1. **Compute Element-wise Difference:** Calculate the difference between predicted and target values: $D = \hat{C} - C_{gt}$ [S5, unofficial].
2. **Square the Difference:** Compute the squared error: $E = D^2$ [S5, unofficial].
3. **Apply Mask (Optional):** If a mask $M$ is provided, apply it to the squared error. For volume rendering, this often involves creating a boolean mask from a thresholded volume or ray validity check [S8, unofficial].
   - Calculate threshold: $T_{thresh} = \text{mask\_threshold\_rel} \cdot \max(V_{mask})$ [S8, unofficial].
   - Create boolean mask: $M = (V_{mask} > T_{thresh})$ [S8, unofficial].
   - Apply mask: $E_{masked} = E \cdot M$ [S8, unofficial].
4. **Flatten Spatial Dimensions:** Reshape the tensor to $[B, -1]$ to flatten height and width (and channels if not averaged separately) [S5, unofficial].
5. **Compute Mean:** Average the squared errors over the batch and spatial dimensions [S5, unofficial].
   - Unmasked: $L_{MSE} = \frac{1}{N} \sum_{i=1}^{N} (\hat{C}_i - C_{gt,i})^2$ [S2, S7, unofficial].
   - Masked: $L_{MSE} = \frac{1}{N_{mask}} \sum_{i \in M} (\hat{C}_i - C_{gt,i})^2$, where $N_{mask}$ is the count of true values in $M$ [S8, unofficial].
6. **Compute PSNR (Optional):** Convert MSE to PSNR for logging [S2, unofficial].
   - If $L_{MSE} > 0$: $L_{PSNR} = 20 \cdot \log_{10}\left(\frac{1}{\sqrt{L_{MSE}}}\right) = -10 \cdot \log_{10}(L_{MSE})$ [S2, S8, unofficial].
   - If $L_{MSE} = 0$: $L_{PSNR} = \infty$ [S8, unofficial].

**Formulas:**
- **MSE:**
  $$ L_{MSE} = \frac{1}{N} \sum_{i=1}^{N} (\hat{C}_i - C_{gt,i})^2 $$
  where $\hat{C}_i$ is the predicted color, $C_{gt,i}$ is the target color, and $N$ is the number of pixels/voxels [S2, S7, unofficial].

- **PSNR:**
  $$ L_{PSNR} = 20 \cdot \log_{10}\left(\frac{1}{\sqrt{L_{MSE}}}\right) $$
  derived from the relationship $PSNR = -10 \cdot \log_{10}(MSE)$ [S2, S8, unofficial].

**Parameters:**
- `mask_threshold_rel`: Relative threshold for mask generation, default value 0.01 [S8, unofficial].
- `batch_size`: Number of random pixels sampled per iteration [S6, unofficial].
- `lr`: Adam learning rate [S6, unofficial].

**Edge Cases and Pitfalls:**
- **Empty Mask:** If the boolean mask has no true values (empty region), raise a `RuntimeError` or handle gracefully to avoid division by zero in masked MSE calculation [S8, unofficial].
- **Zero MSE:** If masked MSE is 0.0, PSNR remains infinity to avoid division by zero in the $\log_{10}$ calculation [S8, unofficial].
- **Precision:** Use float32 or float64 for accumulation to prevent precision loss in large batches [S9, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch

def compute_mse_and_psnr(predicted_rgb, target_rgb, mask=None, mask_threshold_rel=0.01):
    """
    Computes MSE and PSNR between predicted and target RGB images.
    
    Args:
        predicted_rgb: Tensor of shape (B, H, W, 3) or (B, N, 3).
        target_rgb: Tensor of shape (B, H, W, 3) or (B, N, 3).
        mask: Optional boolean tensor of shape (B, H, W) or (B, N).
        mask_threshold_rel: Float, relative threshold for mask generation if mask is not provided.
        
    Returns:
        mse: Scalar tensor.
        psnr: Scalar tensor.
    """
    # Step 1: Compute element-wise difference
    diff = predicted_rgb - target_rgb
    
    # Step 2: Square the difference
    squared_diff = diff ** 2
    
    # Step 3: Apply mask if provided
    if mask is not None:
        # Ensure mask is broadcastable to squared_diff shape
        # If mask is (B, H, W) and squared_diff is (B, H, W, 3), expand mask
        if mask.dim() == 3 and squared_diff.dim() == 4:
            mask = mask.unsqueeze(-1)
        
        # Apply mask
        masked_squared_diff = squared_diff * mask
        
        # Count valid elements
        # Sum over all dimensions except batch to get count per batch item
        # Note: mask.sum() gives total true values. We need per-batch count for mean if batched.
        # For simplicity, assume global mean over all valid elements in batch or per-batch mean.
        # Standard implementation often averages over all valid pixels in the batch.
        valid_count = mask.sum().float()
        
        if valid_count == 0:
            raise RuntimeError("Mask has no true values.")
            
        mse = masked_squared_diff.sum() / valid_count
    else:
        # Step 4: Flatten spatial dimensions
        # Reshape to (B, -1)
        b_size = predicted_rgb.shape[0]
        flat_pred = predicted_rgb.reshape(b_size, -1)
        flat_target = target_rgb.reshape(b_size, -1)
        
        # Step 5: Compute mean
        # Mean over all elements in the batch
        mse = torch.mean((flat_pred - flat_target) ** 2)
        
    # Step 6: Compute PSNR
    if mse > 0:
        psnr = 20.0 * math.log10(1.0 / math.sqrt(mse.item()))
    else:
        psnr = float('inf')
        
    return mse, psnr
```

## Subtopics

### [PSNR Metric Calculation](2742-psnr-metric-calculation.md)
Peak Signal-to-Noise Ratio (PSNR) is a standard benchmark metric used to evaluate the photometric fidelity of reconstructed or rendered images against ground-truth references in differentiable volume rendering pipelines. It quantifies the mean squared error (MSE) between predicted and target pixel values, converting this error into a logarithmic decibel (dB) scale where higher values indicate better quality. Unlike structural metrics such as SSIM, PSNR focuses strictly on pixel-wise intensity differences, making it sensitive to large smooth regions but less responsive to small, high-frequency features like specular highlights.

### [SSIM Metric Calculation](3650-ssim-metric-calculation.md)
SSIM is an image fidelity score used to evaluate the quality of 3D reconstructions and novel 2D views by measuring local structural similarity between a rendered image and a ground truth image. Unlike pixel-wise metrics such as MSE, SSIM captures local patterns of pixel intensities and is correlated with human visual system perception, assuming the human visual system is highly adapted to structured natural scene information. It sets itself apart from its siblings by explicitly modeling luminance, contrast, and structure, whereas PSNR relies solely on global error magnitude and Normal-L2 focuses on geometric normals.

### [Normal-L2 Score Computation](3842-normal-l2-score-computation.md)
The Normal-L2 score evaluates surface geometry by computing the mean squared L2 distance between predicted surface normals and reference normals from a ground-truth mesh. It is computed only on the intersection of predicted and ground-truth masks, distinguishing it from silhouette metrics like iIoU which evaluate mask alignment separately.

### [Minkowski's Metric](4347-minkowski-s-metric.md)
Minkowski's metric is a family of distance measures that treats an image as a 2D signal to calculate the dissimilarity between a reference image and a test image. It generalizes the Mean Squared Error (MSE) by allowing the order parameter $\gamma$ to vary, enabling different sensitivities to pixel-wise errors. Unlike its sibling [PSNR](2742-psnr-metric-calculation.md), which normalizes error by signal power to provide a logarithmic decibel scale, Minkowski's metric provides a raw, unnormalized magnitude of error.

### [Training Loss Computation with Huber and Masking](4765-training-loss-computation-with-huber-and-masking.md)
This method computes the photometric training loss for differentiable volume rendering by combining a Huber loss on RGB values with ray-level validity masking. It addresses the problem of robust optimization in the presence of noisy or saturated pixels, which can destabilize geometry and appearance recovery. It sets itself apart from standard MSE loss [MSE Loss and PSNR Conversion](1651-mse-loss-and-psnr-conversion.md) by using a linear penalty for large errors (reducing outlier sensitivity) and explicitly filtering invalid rays before aggregation.

### [IoU Loss](7110-iou-loss.md)
IoU Loss is a binary segmentation metric used to optimize the occupancy or density fields in differentiable volume rendering by measuring the overlap between predicted and target voxel sets. It differs from pixel-wise photometric losses like [MSE Loss and PSNR Conversion](1651-mse-loss-and-psnr-conversion.md) by operating on binarized volumetric occupancy rather than continuous RGB values, making it robust to intensity ambiguities and suitable for structural consistency.

### [Perceptually-motivated variance metric](9969-perceptually-motivated-variance-metric.md)
The perceptually-motivated variance metric quantifies the error between an 8-bit ambient occlusion (AO) approximation $T$ and a high-fidelity ray-traced reference $R$. It differs from raw pixel Mean Squared Error (MSE) by operating on log-transformed values and spatial gradients, thereby aligning with human visual sensitivity to radiance and its changes. This metric is specifically designed for evaluating AO approximations where reflected ambient radiance is linear in accessibility.

### [Depth Accuracy Metrics](10453-depth-accuracy-metrics.md)
Depth Accuracy Metrics quantify the pixel-wise discrepancy between a predicted depth map $\hat{d}$ and a ground truth depth map $d$. They serve as the primary evaluation standard for depth prior quality in differentiable volume rendering and inverse rendering pipelines. Unlike photometric metrics such as MSE or PSNR which evaluate color fidelity, these metrics assess geometric accuracy. They are distinguished from sibling metrics like SSIM (structural similarity) and Normal-L2 (surface orientation error) by focusing strictly on scalar depth value errors rather than structural coherence or surface normals.

### [Discrete Wavelet Transform Loss](11910-discrete-wavelet-transform-loss.md)
The Discrete Wavelet Transform (DWT) Loss is a regularization term for Neural Radiance Fields (NeRF) that minimizes the difference between the frequency representations of rendered and ground-truth images by decomposing them into sub-bands. It addresses high-frequency overfitting in few-shot NeRF training by explicitly prioritizing low-frequency learning during early stages. Unlike its sibling [MSE Loss and PSNR Conversion](1651-mse-loss-and-psnr-conversion.md), which compares pixel-wise values globally, the DW loss operates on local structures using patch-based sampling to supervise low and mid-frequency components.

### Masked Volume Metrics
A method to exclude empty froxels from evaluation metrics by including only froxels affected by radiance [S10, unofficial]. A froxel is included if any of its RGB channels contain a non-zero value at any point during the test scenario [S10, unofficial]. The effective Region of Interest (ROI) is computed as the union of the scenario's ROI and the ground truth's ROI: $ROI_{effective} = ROI_{scenario} \cup ROI_{truth}$ [S10, unofficial]. This ensures that only relevant froxels are included for each output while excluding completely empty regions [S10, unofficial].

### MSE Bias-Variance Decomposition
Treating the TAA algorithm as a biased estimator to analyze correctness using Mean Squared Error (MSE), decomposed into variance and bias squared [S10, unofficial]. The formula is $MSE[F] = E[(F - G)^2] = V[F] + \beta[F]^2$, where F is the output of the estimator, G is the true value (ground truth), V[F] is the variance of the estimator, and $\beta[F]$ is its bias [S10, unofficial].

### Area Loss Scaling
Scaling the loss of each pixel by the area of that pixel's footprint in the original image to ensure low-resolution pixels have comparable influence to high-resolution pixels in multiscale datasets [S11]. For multiscale datasets, calculate the area scaling factor based on the downsampling factor (e.g., factor of 2 -> area factor 4, factor of 4 -> area factor 16) and multiply the pixel loss by this factor [S11].

### Greedy proposal matching
A fast assignment strategy used during Phase 1 to compute the L2 matching loss between heuristic samples and learnt proposals [S12, unofficial]. For each heuristic sample, find the closest proposal from the learnt proposer set [S12, unofficial]. Optimal matching (Hungarian algorithm) or strict 1-1 greedy matching significantly slows training [S12, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of pixels/voxels, as it involves a single pass to compute differences and averages [S5, S7, unofficial].
- **Memory Complexity:** $O(N)$ to store the intermediate squared difference tensor [S5, unofficial].
- **Performance:** Computationally inexpensive and highly parallelizable on GPUs [S6, unofficial].
- **Strengths:** Simple to implement, differentiable, and widely used as a baseline for photometric fidelity [S3, S6, unofficial].
- **Limitations:** Sensitive to outliers and does not account for structural similarity or human perception [S1, unofficial]. May lead to blurry results compared to perceptual losses [S11].
- **When to use:** When pixel-wise accuracy is the primary goal and structural/semantic information is less critical [S1, unofficial].
- **When not to use:** When structural similarity or perceptual quality is more important, or when dealing with noisy data where robust losses like Huber are preferred [S4765-training-loss-computation-with-huber-and-masking.md].

## Background
The Mean Squared Error (MSE) is a fundamental metric in signal processing and image reconstruction, widely adopted in differentiable volume rendering pipelines for optimizing neural radiance fields (NeRF) and other implicit representations [S1, S2, S3, unofficial]. It was introduced as a standard loss function in early NeRF implementations [S2, S3, unofficial] and has since become a baseline for evaluating photometric fidelity in 3D reconstruction tasks [S1, unofficial]. The conversion to PSNR provides a standardized way to report reconstruction quality across different studies [S2, S3, unofficial].

## Sources
[S1] Differentiable Voxel-based X-ray Rendering Improves Sparse-View 3D CBCT Reconstruction, https://doi.org/10.48550/arxiv.2411.19224, fetched 2026-10-26 (unofficial)
[S2] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S3] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S4] Adaptively Placed Multi-Grid Scene Representation Networks for Large-Scale Data Visualization, https://doi.org/10.1109/tvcg.2023.3327194, fetched 2026-10-26
[S5] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S6] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S7] imanfs/inverse-problems-CT-scans - GitHub, https://github.com/imanfs/inverse-problems-CT-scans, fetched 2026-10-27 (unofficial)
[S8] ValentinKraft/vgs, https://github.com/ValentinKraft/vgs, fetched 2026-10-27 (unofficial)
[S9] hardik0001/ct-reconstruction-gpu, https://github.com/hardik0001/ct-reconstruction-gpu, fetched 2026-10-27 (unofficial)
[S10] Temporal Anti-Aliasing for Real-Time Froxel-Based Volumetric Rendering - Advances in volumetric temporal anti-aliasing for real-time ren dering of participating media, https://openalex.org/W7218067689, fetched 2026-11-01 (unofficial)
[S11] Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00580, fetched 2026-10-26
[S12] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
