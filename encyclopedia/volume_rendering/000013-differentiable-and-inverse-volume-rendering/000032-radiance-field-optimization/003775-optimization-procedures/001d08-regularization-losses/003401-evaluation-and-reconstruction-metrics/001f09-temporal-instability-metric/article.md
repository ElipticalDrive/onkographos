# Temporal Instability Metric

Also called: Flicker Metric, TI Metric, Temporal Flicker, Temporal Variation in Reconstruction Error

## Overview
The Temporal Instability (TI) metric quantifies the temporal coherence of a volume rendering or novel view synthesis method by measuring the frame-to-frame variation in reconstruction error [S1, unofficial]. It serves as a diagnostic for visual flickering artifacts, which are often imperceptible in static per-frame metrics like RMSE or PSNR [S1, unofficial]. Unlike the feature-space Flicker metric, which compares temporal changes in feature representations, the TI metric operates directly in the froxel (voxel) space, measuring the variance of the error signal over time [S1, unofficial]. It is distinct from structural similarity metrics as it focuses on signal stability rather than perceptual similarity [S1, unofficial].

## How it works
The TI metric evaluates the stability of the reconstruction error across a sequence of frames. It identifies regions where the error oscillates significantly, which correlates with visual flicker [S1, unofficial].

### Inputs and Outputs
- **Inputs**:
    - $I(t)$: Reconstructed volume or rendered frames at time $t$.
    - $I^*(t)$: Ground truth volume or frames at time $t$.
    - $\Omega$: Region of interest (ROI) within the volume or image space.
    - $T$: Total number of frames in the sequence.
- **Outputs**:
    - $TI$: A scalar value representing the root mean square of the temporal error changes. Lower values indicate higher temporal stability.

### Data Structures
- **Error Field $E(t)$**: A 3D array (for volumes) or 2D array (for images) of the same spatial dimensions as the input, where each element $E_i(t)$ stores the reconstruction error for froxel/pixel $i$ at frame $t$.
    - Layout: $E \in \mathbb{R}^{H \times W \times D}$ (or $\mathbb{R}^{H \times W}$).
    - Values: Component-wise error (e.g., squared difference per color channel).

### Steps
1.  **Compute Per-Frame Error**: For each frame $t$ in the sequence $[1, T]$, calculate the error field $E(t)$ between the reconstruction $I(t)$ and ground truth $I^*(t)$.
    - The error is typically computed component-wise (e.g., per RGB channel).
2.  **Compute Temporal Error Differences**: For each froxel $i \in \Omega$ and frame $t > 1$, calculate the squared difference between consecutive error values: $(E_i(t) - E_i(t-1))^2$.
3.  **Aggregate Over Time**: For each froxel $i$, average the squared temporal differences over the $T$ frames.
4.  **Aggregate Over Space**: Average the temporal results over all froxels $i$ in the region of interest $\Omega$.
5.  **Finalize**: Take the square root of the spatial average to obtain the TI metric. Color channels are averaged at the very end to produce the final scalar value [S1, unofficial].

### Formula
The Temporal Instability metric is defined as [S1, unofficial]:

$$
TI = \sqrt{\frac{1}{|\Omega|} \sum_{i \in \Omega} \frac{1}{T} \sum_{t=2}^{T} (E_i(t) - E_i(t-1))^2}
$$

Where:
- $|\Omega|$ is the number of froxels in the region of interest.
- $i$ indexes the froxels in $\Omega$.
- $T$ is the total number of frames.
- $E_i(t)$ is the error of froxel $i$ at frame $t$.
- All operations are component-wise, and color channels are averaged at the end [S1, unofficial].

### Parameters
- **Region of Interest ($\Omega$)**: The spatial subset of the volume/image to evaluate. Not given in the sources.
- **Percentile Reporting**: The 95th percentile of the TI signal is sometimes presented instead of the mean to omit large one-frame outliers and align better with visually perceived flicker [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_temporal_instability(recon_frames, gt_frames, region_of_interest):
    """
    Computes the Temporal Instability (TI) metric.
    
    Args:
        recon_frames: List of reconstructed frames/volumes [I(1), ..., I(T)].
        gt_frames: List of ground truth frames/volumes [I*(1), ..., I*(T)].
        region_of_interest: Boolean mask or indices for the spatial region Omega.
        
    Returns:
        ti_value: Scalar TI metric.
        ti_95: 95th percentile of the TI signal.
    """
    T = len(recon_frames)
    if T < 2:
        return 0.0, 0.0

    # Initialize error difference accumulator
    # Shape matches the spatial dimensions of the frames
    error_diff_sq_sum = np.zeros_like(recon_frames[0], dtype=np.float32)
    
    # Compute component-wise error differences between consecutive frames
    for t in range(1, T):
        # Compute error field E(t) and E(t-1)
        # Assuming MSE per component: (recon - gt)^2
        err_t = (recon_frames[t] - gt_frames[t]) ** 2
        err_t_minus_1 = (recon_frames[t-1] - gt_frames[t-1]) ** 2
        
        # Compute squared difference of errors: (E(t) - E(t-1))^2
        # Note: The formula specifies (E_i(t) - E_i(t-1))^2. 
        # If E is already squared error, this is ((recon-gt)^2_t - (recon-gt)^2_{t-1})^2
        diff = err_t - err_t_minus_1
        error_diff_sq_sum += diff ** 2

    # Average over time T
    # The formula divides by T. Note: sum is from t=2 to T, so there are T-1 terms.
    # However, the source formula uses 1/T. We follow the source formula exactly.
    avg_error_diff_sq = error_diff_sq_sum / T

    # Average over spatial region Omega
    if region_of_interest is not None:
        masked_avg = np.mean(avg_error_diff_sq[region_of_interest])
    else:
        masked_avg = np.mean(avg_error_diff_sq)

    # Average color channels at the end
    # Assuming last dimension is color (C)
    if masked_avg.ndim > 1:
        # If masked_avg is 2D (HxW) or 3D (HxWxD), we need to handle channels
        # The source says "color channels averaged at the very end"
        # If avg_error_diff_sq has shape (H, W, C), mean over H,W gives (C,)
        # Then mean over C gives scalar.
        # If region_of_interest reduces spatial dims, we still have C dims.
        ti_value = np.mean(masked_avg)
    else:
        ti_value = np.mean(masked_avg)

    # Compute 95th percentile of the TI signal
    # The "TI signal" refers to the per-froxel temporal instability values before spatial averaging
    # Calculate per-froxel TI
    per_froxel_ti_sq = avg_error_diff_sq # Shape (H, W, C) or (H, W, D, C)
    
    # Flatten spatial dimensions to get a list of per-froxel values
    # We need to average channels for each froxel first to get a scalar per froxel?
    # Source: "All operations are done component-wise and the color channels averaged at the very end"
    # This implies the per-froxel signal might be multi-channel. 
    # For percentile, we likely flatten all spatial and channel dims to get a distribution of values.
    signal_values = per_froxel_ti_sq.flatten()
    ti_95 = np.percentile(signal_values, 95)

    return np.sqrt(ti_value), np.sqrt(ti_95)
```

### Edge Cases and Pitfalls
- **Ground Truth Changes**: The TI metric does not distinguish between rapid changes in the ground truth and slow responses in the reconstructed signal [S1, unofficial]. A slow-reacting algorithm may produce a high TI spike even if the visual output is stable [S1, unofficial].
- **Visual Correlation**: High visual flicker implies high TI, but high TI does not necessarily imply visual flicker [S1, unofficial].
- **Outliers**: Large one-frame outliers can skew the mean TI. Using the 95th percentile is recommended to mitigate this [S1, unofficial].

## Operating characteristics
- **Complexity**: Time complexity is $O(N \cdot T)$, where $N$ is the number of froxels in $\Omega$ and $T$ is the number of frames. Memory complexity is $O(N)$ to store the error difference accumulator.
- **Performance**: On the Neural3DVideo dataset (2 input views), the full model achieved a flicker score of 2.916, compared to 5.714 for GS-LRM and 2.746 for 4DGS [S2]. On LongVolumetricVideo (4 input views), the full model achieved 7.782, compared to 8.463 for GS-LRM and 6.528 for 4DGS [S2].
- **Strengths**: Directly measures signal variance in the reconstruction error, providing a quantitative link to temporal stability [S1, unofficial].
- **Limitations**: Does not account for perceptual quality or structural similarity [S1, unofficial]. May not correlate perfectly with human perception of flicker due to ground truth dynamics [S1, unofficial].

## Background
The Temporal Instability metric was introduced in the context of volumetric temporal anti-aliasing (TAA) for real-time rendering of participating media [S1, unofficial]. It was designed to complement RMSE by capturing temporal artifacts that static metrics miss [S1, unofficial]. A related "Flicker Metric" was later introduced in the context of novel view streaming for dynamic scene reconstruction, using feature-space variations from a VGG network [S2].

## Sources
[S1] Temporal Anti-Aliasing for Real-Time Froxel-Based Volumetric Rendering - Advances in volumetric temporal anti-aliasing for real-time ren dering of participating media, https://openalex.org/W7218067689, fetched 2026-11-01 (unofficial)
[S2] Learning Efficient Fuse-and-Refine for Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2503.14698v2, fetched 2026-10-27
