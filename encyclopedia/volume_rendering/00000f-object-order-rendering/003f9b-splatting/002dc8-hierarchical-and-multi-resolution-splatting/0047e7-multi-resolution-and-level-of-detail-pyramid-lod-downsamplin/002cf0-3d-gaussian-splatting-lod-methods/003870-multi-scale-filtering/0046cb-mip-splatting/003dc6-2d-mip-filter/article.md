# 2D Mip Filter

Also called: Mip-Splatting 2D filter, View-dependent 2D Gaussian filter

## Overview
The 2D Mip Filter mitigates aliasing artifacts in 3D Gaussian Splatting when rendering from camera distances or focal lengths larger than those encountered during training [S1, unofficial]. It extends the standard 2D Gaussian filter by introducing a view-dependent opacity compensation factor that prevents excessive blurring as Gaussians become smaller relative to the pixel grid [S1, unofficial]. Unlike the basic 2D Gaussian filter which dilates the covariance matrix uniformly, this method dynamically adjusts opacity based on the ratio of the original to the dilated covariance determinant [S1, unofficial].

## How it works
The method modifies the projected 2D covariance matrix and the opacity of each Gaussian during the rendering pass. It is typically combined with a 3D smoothing filter, which requires a smaller base variance for the 2D filter to avoid over-smoothing [S1, unofficial].

**Inputs:**
*   $\Sigma_{2D}$: The projected 2D covariance matrix of a Gaussian.
*   $o$: The original opacity of the Gaussian.
*   $\kappa_{3D}$: The variance parameter for the 2D filter dilation.

**Outputs:**
*   $\hat{\Sigma}_{2D}$: The dilated covariance matrix used for rasterization.
*   $\hat{o}$: The compensated opacity used for alpha blending.

**Data Structures:**
*   $\Sigma_{2D}$: A $2 \times 2$ symmetric matrix.
*   $\mathbf{I}$: The $2 \times 2$ identity matrix.

**Parameters:**
*   $\kappa_{3D}$: The added variance. Default value is $0.1$ when combined with the 3D smoothing filter [S1, unofficial]. (Note: The original 3DGS uses $0.3$ for the 2D filter alone, but this value is reduced in the Mip-Splatting context) [S1, unofficial].

**Steps:**
1.  **Dilate Covariance:** Add the variance parameter to the diagonal of the projected covariance matrix to prevent aliasing from undersampling:
    $$ \hat{\Sigma}_{2D} = \Sigma_{2D} + \kappa_{3D} \mathbf{I} $$
2.  **Compute Compensation Factor:** Calculate the square root of the ratio of the determinants of the original and dilated covariance matrices. This factor represents the relative volume change:
    $$ \text{factor} = \sqrt{\frac{|\Sigma_{2D}|}{|\hat{\Sigma}_{2D}|}} $$
3.  **Compensate Opacity:** Multiply the original opacity by the compensation factor to maintain visual consistency at different scales:
    $$ \hat{o} = \text{factor} \cdot o $$
4.  **Bounding Box Update:** Use the compensated opacity $\hat{o}$ (rather than the original $o$) when computing opacity-aware bounding boxes for culling [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_2d_mip_filter(Sigma_2D: np.ndarray, opacity: float, kappa_3D: float = 0.1) -> tuple:
    """
    Computes the dilated covariance and compensated opacity for the 2D Mip Filter.
    
    Args:
        Sigma_2D: 2x2 numpy array representing the projected 2D covariance matrix.
        opacity: Original opacity scalar.
        kappa_3D: Variance parameter for dilation (default 0.1).
        
    Returns:
        Tuple of (dilated_Sigma_2D, compensated_opacity)
    """
    # Step 1: Dilate covariance matrix
    # Add kappa_3D to the diagonal elements
    dilated_Sigma_2D = Sigma_2D + kappa_3D * np.eye(2)
    
    # Step 2: Compute determinants
    det_original = np.linalg.det(Sigma_2D)
    det_dilated = np.linalg.det(dilated_Sigma_2D)
    
    # Guard against numerical instability if determinant is near zero or negative
    # [G] Clamp determinants to a small positive epsilon to avoid NaNs in sqrt/log
    epsilon = 1e-8
    det_original = max(det_original, epsilon)
    det_dilated = max(det_dilated, epsilon)
    
    # Step 3: Compute compensation factor
    # factor = sqrt(|Sigma_2D| / |Sigma_hat_2D|)
    factor = np.sqrt(det_original / det_dilated)
    
    # Step 4: Compensate opacity
    compensated_opacity = factor * opacity
    
    return dilated_Sigma_2D, compensated_opacity
```

**Edge Cases and Pitfalls:**
*   **Gradient Instability:** The analytical derivatives of the opacity compensation factor with respect to $\Sigma_{2D}$ are numerically unstable and can cause gradients to explode during training [S1, unofficial].
*   **Bounding Box Accuracy:** Failing to use the compensated opacity $\hat{o}$ for bounding box calculations can lead to incorrect culling decisions, potentially rendering Gaussians that should be culled or missing those that should be included [S1, unofficial].

## Subtopics

### 2D Mip Filter Gradient Stabilization
A technique to prevent exploding gradients in the 2D Mip filter by detaching the compensation factor from the gradient computation for the 2D covariance matrix $\Sigma_{2D}$ [S1, unofficial]. The original implementation [98] frequently clips extreme values, while other re-implementations [56, 65] use non-analytical gradients that do not match the analytical derivative [S1, unofficial]. Detaching the factor ensures stable gradients while maintaining the analytical form of the forward pass [S1, unofficial]. A stable implementation for the full analytical derivative is optionally available but not recommended for standard training [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The additional computation involves determinant calculation and square root operations per Gaussian, which is negligible compared to the rasterization cost. The authors report that their simplifications effectively eliminate any training overhead compared to the original implementation [S1, unofficial].
*   **Memory Complexity:** No additional memory is required beyond the stored Gaussian parameters.
*   **Performance:** On the Mip-NeRF360 dataset, enabling full anti-aliasing (including the 2D Mip Filter) improves PSNR at 0.5x resolution from 25.11 to 28.39, while maintaining comparable PSNR at 1x resolution (27.56 vs 27.54) [S1, unofficial].
*   **Strengths:** Significantly reduces aliasing artifacts when rendering at resolutions different from training resolution or from distant cameras [S1, unofficial].
*   **Limitations:** Can lead to increasingly blurry renderings if the compensation factor is not applied, as the dilated covariance dominates at large distances [S1, unofficial].

## Background
The 2D Mip Filter was introduced by Yu et al. as part of the Mip-Splatting framework [S1, unofficial]. It builds upon the 2D Gaussian filter used in the original 3D Gaussian Splatting (3DGS) by Kerbl et al. [37], which adds a constant variance to prevent aliasing when Gaussians fall between pixels [S1, unofficial]. Yu et al. identified that this uniform dilation causes excessive blurring during inference when the camera is far away, and introduced the view-dependent opacity compensation to mitigate this [S1, unofficial]. The method is cited as [98] in the Faster-GS paper [S1, unofficial].

## Sources
[S1] Faster-GS: Analyzing and Improving Gaussian Splatting Optimization, https://arxiv.org/abs/2602.09999v1, fetched 2026-10-26 (unofficial)
