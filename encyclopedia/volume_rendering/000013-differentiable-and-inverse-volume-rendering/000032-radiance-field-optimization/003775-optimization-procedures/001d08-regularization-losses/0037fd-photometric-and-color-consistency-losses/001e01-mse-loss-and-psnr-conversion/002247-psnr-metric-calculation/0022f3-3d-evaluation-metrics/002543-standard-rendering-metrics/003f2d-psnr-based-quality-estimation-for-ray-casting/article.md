# PSNR-based Quality Estimation for Ray Casting

Also called: Noise-based quality estimation, PSNR for DVR quality

## Overview
This method provides an objective, numerical metric for estimating sampling artifacts in Direct Volume Rendering (DVR) by treating rendered images as noisy signals [S1, unofficial]. Unlike traditional assessment which relies on subjective human visual judgment, it quantifies the dispersion of pixel colors across multiple renders with different stochastic jitter seeds to produce a Peak Signal-to-Noise Ratio (PSNR) value [S1, unofficial]. It is distinct from standard PSNR calculations for image compression because it measures the variance of the rendering algorithm itself rather than the error between a rendered image and a ground-truth reference [S1, unofficial].

## How it works
**Inputs:**
1.  A volumetric dataset.
2.  A fixed camera viewpoint and transfer function.
3.  A set of $K$ rendered images of the same scene, generated using the same ray casting algorithm but with different random seeds for stochastic jittering of ray starting positions [S1, unofficial].
4.  A binary mask or logic to identify background pixels (pixels with null dispersion) [S1, unofficial].

**Outputs:**
1.  A global PSNR value (in dB) representing the overall rendering quality [S1, unofficial].
2.  Optionally, a per-pixel quality map showing local sampling artifact intensity [S1, unofficial].

**Data Structures:**
*   **Rendered Images:** A list of $K$ images, each of dimensions $n \times m$ (width $\times$ height). Each pixel contains a color vector $C$.
*   **Color Space:** The color values are converted to the YPbPr color space before calculation [S1, unofficial].
*   **Background Mask:** A boolean grid of size $n \times m$ indicating which pixels are background (null dispersion) [S1, unofficial].

**Parameters:**
*   $n, m$: Image width and height [S1, unofficial].
*   $K$: Number of rendered images with different jitter seeds [S1, unofficial].
*   $MAX_C$: Maximum possible length of the color vector. Set to $1$ in the source implementation [S1, unofficial].
*   $N$: Number of non-background pixels in the image [S1, unofficial].

**Steps:**
1.  **Render:** Generate $K$ images of the volumetric dataset from the same viewpoint using the same transfer function and visualization settings, varying only the random seed for stochastic jittering of ray starting positions [S1, unofficial].
2.  **Convert:** Convert the color values of all $K$ rendered images to the YPbPr color space [S1, unofficial].
3.  **Calculate Per-Pixel Error:** For each pixel $(i, j)$, calculate the dispersion (error) of the color vector.
    *   Let $C_k(i, j)$ be the color vector of pixel $(i, j)$ in the $k$-th rendered image.
    *   Calculate the average color $M(i, j)$ across all $K$ images:
        $$ M(i, j) = \frac{1}{K} \sum_{k=1}^{K} C_k(i, j) $$
    *   Calculate the error (dispersion) $D(i, j)$ for pixel $(i, j)$:
        $$ D(i, j) = \frac{1}{K} \sum_{k=1}^{K} \| M(i, j) - C_k(i, j) \|^2 $$
        *Note: The source formula is written as $M(M(C(i, j)) - C(i, j))^2$. In the context of dispersion of a random variable, this represents the variance. The outer $M$ denotes the mean operator applied to the squared difference. The norm $\| \cdot \|$ is applied to the color vector difference [S1, unofficial].*
4.  **Calculate MSE:** Compute the Mean-Square Error (MSE) over all non-background pixels.
    *   Identify the set of indices $\Omega$ for all pixels $(i, j)$ that are not background (i.e., have non-null dispersion) [S1, unofficial].
    *   Calculate $N = |\Omega|$ [S1, unofficial].
    *   Calculate MSE:
        $$ MSE = \frac{1}{N} \sum_{(i, j) \in \Omega} D(i, j) $$
    *   *Pitfall:* Background pixels have null dispersion. Including them in the sum would artificially inflate the quality score (overstatement) [S1, unofficial].
5.  **Calculate PSNR:** Compute the global PSNR in decibels:
    $$ PSNR = 10 \log_{10} \left( \frac{MAX_C^2}{MSE} \right) $$
    With $MAX_C = 1$ [S1, unofficial].
6.  **Optional Quality Map:** To visualize local quality, calculate the PSNR for each single pixel individually using the local dispersion $D(i, j)$:
    $$ PSNR(i, j) = 10 \log_{10} \left( \frac{MAX_C^2}{D(i, j)} \right) $$
    (Handle division by zero if $D(i, j) = 0$ by assigning infinite PSNR or a maximum value [G]).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def calculate_psnr_quality(rendered_images: list[np.ndarray], background_mask: np.ndarray) -> float:
    """
    Calculate PSNR-based quality estimation for ray casting.
    
    Args:
        rendered_images: List of K numpy arrays, each of shape (n, m, 3) in YPbPr space.
        background_mask: Boolean numpy array of shape (n, m), True for background pixels.
        
    Returns:
        Global PSNR in dB.
    """
    if not rendered_images:
        return 0.0
        
    # Stack images into a single array of shape (K, n, m, 3)
    images = np.stack(rendered_images, axis=0)
    K, n, m, _ = images.shape
    
    # Calculate mean color across the K renders for each pixel
    # Mean color M(i,j) shape: (n, m, 3)
    mean_color = np.mean(images, axis=0)
    
    # Calculate dispersion (variance) for each pixel
    # D(i,j) = mean(||M - C_k||^2)
    # Difference shape: (K, n, m, 3)
    diff = mean_color - images
    
    # Squared Euclidean norm for each pixel across the K samples
    # Sum over color channels (axis=-1) then average over K (axis=0)
    # Note: np.mean over axis=0 of squared diffs gives the variance
    squared_diff = np.square(diff)
    dispersion = np.mean(squared_diff, axis=0) # Shape: (n, m)
    
    # Identify non-background pixels
    # background_mask is True for background. We want False (non-background).
    non_bg_mask = ~background_mask
    
    # Extract dispersion values for non-background pixels
    # Flatten dispersion to 1D for easy indexing
    dispersion_flat = dispersion.flatten()
    non_bg_mask_flat = non_bg_mask.flatten()
    
    non_bg_dispersion = dispersion_flat[non_bg_mask_flat]
    
    N = len(non_bg_dispersion)
    
    if N == 0:
        # No non-background pixels, quality is undefined or infinite
        return float('inf')
        
    # Calculate MSE
    mse = np.mean(non_bg_dispersion)
    
    # Calculate PSNR
    MAX_C = 1.0
    if mse == 0:
        return float('inf')
        
    psnr = 10 * np.log10((MAX_C ** 2) / mse)
    
    return psnr
```

## Operating characteristics
*   **Time Complexity:** $O(K \cdot n \cdot m)$, where $K$ is the number of renders, and $n, m$ are image dimensions. The method requires rendering the scene $K$ times, which is the dominant cost. The post-processing calculation is linear in the number of pixels and renders.
*   **Memory Complexity:** $O(K \cdot n \cdot m)$ to store the $K$ rendered images.
*   **Strengths:** Provides an objective, numerical metric that correlates with perceptual noise levels. It allows for automated comparison of different Ray Casting (RC) algorithms and sampling rates without human intervention [S1, unofficial]. It generates quality maps to identify specific areas of high/low rendering quality [S1, unofficial].
*   **Limitations:** It is insensitive to perceptual quality aspects other than noise (e.g., color accuracy, structural fidelity) [S1, unofficial]. It requires multiple renders with different jitter seeds, increasing computational cost. It assumes that sampling artifacts manifest as noise, which may not hold for all types of rendering errors.
*   **When to use:** When comparing different RC algorithms (e.g., unoptimized vs. pre-integrated) or tuning sampling rates to achieve a desired quality-performance trade-off [S1, unofficial].
*   **When not to use:** When a ground-truth reference image is available (standard PSNR against reference is more appropriate) or when perceptual quality metrics (SSIM, LPIPS) are required.

## Background
The method was proposed by N. Gavrilov and N. Gavrilov in 2013 [S1, unofficial]. It addresses the lack of numerical quality estimation approaches for images obtained by visualization of medical and scientific volumetric datasets [S1, unofficial]. Prior to this, researchers primarily relied on subjective human visual judgment, placing images side-by-side for comparison [S1, unofficial]. The method leverages the fact that sampling artifacts in DVR, caused by uniform sampling missing thin features, can be converted to noise via stochastic jittering [S1, unofficial]. It adapts the PSNR metric, traditionally used for image compression error measurement, to quantify the noise introduced by the rendering algorithm itself [S1, unofficial].

## Sources
[S1] Volume Ray Casting quality estimation in terms of Peak Signal-to-Noise Ratio, https://doi.org/10.2312/conf/eg2013/posters/007-008, fetched 2026-10-26 (unofficial)
