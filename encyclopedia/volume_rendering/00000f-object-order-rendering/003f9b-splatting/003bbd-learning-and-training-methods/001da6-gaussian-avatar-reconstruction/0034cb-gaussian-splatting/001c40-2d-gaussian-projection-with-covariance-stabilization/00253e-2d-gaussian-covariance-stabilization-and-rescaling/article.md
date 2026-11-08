# 2D Gaussian Covariance Stabilization and Rescaling

Also called: Conic Matrix and Rescale Factor, Covariance Rescaling, Gaussian Covariance Pre-filtering, Pixel-spanning Gaussian

## Overview
This method stabilizes the screen-space projection of 3D Gaussians by applying a low-pass filter to the 2D covariance matrix, ensuring that each Gaussian spans at least the width of a single pixel to reduce aliasing artifacts [S2, unofficial]. It computes the inverse covariance (conic) matrix and a corresponding rescale factor that preserves the integral of the Gaussian function after the covariance modification [S1, unofficial]. It builds on [2D Gaussian Projection with Covariance Stabilization](1198-2d-gaussian-projection-with-covariance-stabilization.md) by adding explicit anti-aliasing rescaling rather than just numerical stabilization [S2, unofficial].

## How it works
**Inputs:**
- $\Sigma_n^{2D}$: The $2 \times 2$ screen-space covariance matrix for the $n$-th Gaussian, computed from the 3D covariance and projection Jacobian.
- $o_n$: The opacity of the $n$-th Gaussian.
- $s$: The anti-aliasing hyper-parameter (default 0.3) [S2, unofficial].

**Outputs:**
- `conic_and_rescale`: A `vec4` containing the upper-triangular elements of the inverse covariance matrix and the rescale factor: $[ \Sigma^{-1}_{00}, \Sigma^{-1}_{01}, \Sigma^{-1}_{11}, \text{rescale} ]$.

**Data Structures:**
- The covariance matrix $\Sigma$ is stored as a $2 \times 2$ symmetric matrix.
- The output is packed into a 4-component vector to minimize memory bandwidth during rasterization.

**Steps:**
1. Compute the determinant of the original covariance matrix:
   $$ \det_{\text{pre}} = \det(\Sigma_n^{2D}) $$
2. Apply the low-pass filter by adding the scalar $s$ to the diagonal elements of the covariance matrix:
   $$ \Sigma_{\text{filtered}} = \Sigma_n^{2D} + s \cdot \mathbf{I} $$
   where $\mathbf{I}$ is the $2 \times 2$ identity matrix [S2, unofficial].
3. Compute the determinant of the filtered covariance matrix:
   $$ \det_{\text{post}} = \det(\Sigma_{\text{filtered}}) $$
4. Compute the rescale factor to preserve the integral (area) of the Gaussian distribution:
   $$ \text{rescale} = \sqrt{\max(0.0, \frac{\det_{\text{pre}}}{\det_{\text{post}}})} $$
   The $\max(0.0, \dots)$ guard handles potential numerical noise where $\det_{\text{pre}}$ might be slightly negative due to precision errors [S1, unofficial].
5. Compute the inverse of the filtered covariance matrix (the conic matrix) analytically for a $2 \times 2$ matrix:
   $$ \Sigma^{-1}_{\text{filtered}} = \frac{1}{\det_{\text{post}}} \begin{bmatrix} \Sigma_{\text{filtered},11} & -\Sigma_{\text{filtered},01} \\ -\Sigma_{\text{filtered},10} & \Sigma_{\text{filtered},00} \end{bmatrix} $$
   Since $\Sigma$ is symmetric, $\Sigma_{01} = \Sigma_{10}$ [S1, unofficial].
6. Pack the result into a `vec4`:
   $$ \text{conic\_and\_rescale} = [\Sigma^{-1}_{\text{filtered},00}, \Sigma^{-1}_{\text{filtered},01}, \Sigma^{-1}_{\text{filtered},11}, \text{rescale}] $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_conic_and_rescale(cov_2d, opacity, s=0.3):
    """
    Computes the conic matrix and rescale factor for a 2D Gaussian.
    
    Args:
        cov_2d: 2x2 numpy array or tensor representing Sigma_n^2D.
        opacity: Scalar opacity o_n.
        s: Anti-aliasing scalar (default 0.3).
        
    Returns:
        conic_rescale: vec4 [inv_cov_00, inv_cov_01, inv_cov_11, rescale]
    """
    # Step 1: Pre-filter determinant
    det_pre = cov_2d[0, 0] * cov_2d[1, 1] - cov_2d[0, 1] * cov_2d[1, 0]
    
    # Step 2: Apply low-pass filter (add s to diagonal)
    cov_filtered = cov_2d.copy()
    cov_filtered[0, 0] += s
    cov_filtered[1, 1] += s
    
    # Step 3: Post-filter determinant
    det_post = cov_filtered[0, 0] * cov_filtered[1, 1] - cov_filtered[0, 1] * cov_filtered[1, 0]
    
    # Step 4: Compute rescale factor
    # Guard against negative determinant due to numerical precision
    ratio = det_pre / det_post
    if ratio < 0:
        ratio = 0.0
    rescale = sqrt(ratio)
    
    # Step 5: Compute inverse covariance (conic) analytically
    # inv = (1/det) * [[d, -b], [-c, a]]
    inv_00 = cov_filtered[1, 1] / det_post
    inv_01 = -cov_filtered[0, 1] / det_post
    inv_11 = cov_filtered[0, 0] / det_post
    
    # Step 6: Return packed vec4
    return [inv_00, inv_01, inv_11, rescale]
```

**Parameters:**
- $s$: The smoothing scalar added to the covariance diagonal. Default value is 0.3 [S2, unofficial]. This value ensures the Gaussian spans approximately one pixel width.

**Edge Cases and Pitfalls:**
- **Negative Determinant:** If the original covariance matrix is numerically unstable and yields a negative determinant, the ratio in the rescale factor calculation becomes negative. The $\max(0.0, \dots)$ operation prevents NaN propagation [S1, unofficial].
- **Degenerate Gaussians:** If the Gaussian is extremely thin or flat, the determinant may be close to zero. The addition of $s \cdot \mathbf{I}$ prevents division by zero in the inverse calculation [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per Gaussian. The operations involve a fixed number of arithmetic operations (determinants, additions, square root, division).
- **Memory Complexity:** $O(1)$ additional memory per Gaussian for the filtered covariance and intermediate determinants.
- **Strengths:** Effectively reduces high-frequency aliasing artifacts in rendered images by ensuring Gaussians are not smaller than a pixel [S2, unofficial]. The rescale factor preserves the total energy (opacity integral) of the Gaussian, preventing brightness loss when smoothing [S1, unofficial].
- **Limitations:** The fixed scalar $s$ is a heuristic. In scenes with extreme depth variations or very high resolution, a single scalar may not optimally balance anti-aliasing and sharpness [S2, unofficial]. It adds a small computational overhead compared to naive projection.

## Background
This technique is part of the Gaussian Splatting rendering pipeline, specifically addressing the aliasing issues inherent in rasterizing point-based primitives [S2, unofficial]. The concept of pre-filtering Gaussians to match the pixel footprint is rooted in signal processing principles for anti-aliasing [S2, unofficial]. The specific implementation using a scalar addition to the covariance diagonal and a determinant-based rescale factor is documented in the `gsplat` library and related implementations [S1, unofficial][S2, unofficial].

## Sources
[S1] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
[S2] gsplat: An Open-Source Library for Gaussian Splatting, https://doi.org/10.48550/arXiv.2409.06765, fetched 2026-10-29 (unofficial)
