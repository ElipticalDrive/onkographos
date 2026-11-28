# 2D Gaussian Probability Density (Standard)

Also called: 2D Gaussian PDF

## Overview
Computes the normalized probability density function (PDF) value of a 2D Gaussian at a query point `xy`, given its mean and covariance matrix [S1, unofficial]. It serves as the fundamental evaluation primitive for Gaussian splatting rasterization [G].

## How it works
**Inputs:**
- `xy`: A 2D point $(x, y)$ in the projected image plane, represented as a vector $\mathbf{xy} = [x, y]^T$ [S1, unofficial].
- `gaussian_mean`: The 2D mean position $\boldsymbol{\mu} = [\mu_x, \mu_y]^T$ of the Gaussian primitive [S1, unofficial].
- `gaussian_covariance`: A $2 \times 2$ symmetric positive-definite covariance matrix $\Sigma = \begin{bmatrix} \Sigma_{00} & \Sigma_{01} \\ \Sigma_{10} & \Sigma_{11} \end{bmatrix}$, where $\Sigma_{01} = \Sigma_{10}$ [S1, unofficial].

**Outputs:**
- `pdf`: The scalar probability density value $P(\mathbf{xy})$ [S1, unofficial].

**Data Structures:**
- `xy`: `float32` or `float64` vector of size 2 [G].
- `gaussian_mean`: `float32` or `float64` vector of size 2 [G].
- `gaussian_covariance`: `float32` or `float64` matrix of shape $(2, 2)$ [G].

**Steps:**
1. Compute the difference vector between the query point and the mean:
   $$ \Delta \mathbf{x} = \mathbf{xy} - \boldsymbol{\mu} $$
   Let $\Delta x = \Delta \mathbf{x}[0]$ and $\Delta y = \Delta \mathbf{x}[1]$ [G].

2. Compute the determinant of the covariance matrix $\Sigma$:
   $$ \det(\Sigma) = \Sigma_{00}\Sigma_{11} - \Sigma_{01}\Sigma_{10} $$
   Since $\Sigma$ is symmetric, $\Sigma_{01} = \Sigma_{10}$, so $\det(\Sigma) = \Sigma_{00}\Sigma_{11} - \Sigma_{01}^2$ [G].

3. Compute the inverse of the $2 \times 2$ covariance matrix $\Sigma^{-1}$ analytically:
   $$ \Sigma^{-1} = \frac{1}{\det(\Sigma)} \begin{bmatrix} \Sigma_{11} & -\Sigma_{01} \\ -\Sigma_{10} & \Sigma_{00} \end{bmatrix} $$
   Let $\Sigma^{-1} = \begin{bmatrix} C_{00} & C_{01} \\ C_{10} & C_{11} \end{bmatrix}$ [G]. Note that $C_{01} = C_{10} = -\Sigma_{01} / \det(\Sigma)$ [G].

4. Compute the quadratic form $\Delta \mathbf{x}^T \Sigma^{-1} \Delta \mathbf{x}$:
   $$ Q = \Delta \mathbf{x}^T \Sigma^{-1} \Delta \mathbf{x} = C_{00} \Delta x^2 + 2 C_{01} \Delta x \Delta y + C_{11} \Delta y^2 $$
   Alternatively, using matrix multiplication: $\mathbf{v} = \Sigma^{-1} \Delta \mathbf{x}$, then $Q = \mathbf{v} \cdot \Delta \mathbf{x}$ [S1, unofficial].

5. Compute the exponent:
   $$ E = -0.5 \cdot Q $$
   [S1, unofficial]

6. Compute the normalization constant:
   $$ Z = 2 \pi \sqrt{\det(\Sigma)} $$
   [G]

7. Return the PDF value:
   $$ P(\mathbf{xy}) = \frac{\exp(E)}{Z} $$
   [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_2d_gaussian_pdf(xy, gaussian_mean, gaussian_covariance):
    """
    Computes the normalized 2D Gaussian PDF value.
    
    Args:
        xy: List or array of [x, y] coordinates of the query point.
        gaussian_mean: List or array of [mu_x, mu_y] coordinates of the Gaussian mean.
        gaussian_covariance: 2x2 matrix [[cov00, cov01], [cov10, cov11]].
        
    Returns:
        float: The probability density value.
    """
    # Step 1: Compute mean difference
    dx = xy[0] - gaussian_mean[0]
    dy = xy[1] - gaussian_mean[1]
    
    # Step 2: Compute determinant
    cov00 = gaussian_covariance[0][0]
    cov01 = gaussian_covariance[0][1]
    cov10 = gaussian_covariance[1][0]
    cov11 = gaussian_covariance[1][1]
    
    det_cov = cov00 * cov11 - cov01 * cov10
    
    # Handle degenerate cases where determinant is zero or negative
    if det_cov <= 0:
        return 0.0
        
    # Step 3: Compute inverse covariance matrix elements
    inv_det = 1.0 / det_cov
    C00 = cov11 * inv_det
    C01 = -cov01 * inv_det
    C10 = -cov10 * inv_det
    C11 = cov00 * inv_det
    
    # Step 4: Compute quadratic form dx^T * inv_cov * dx
    quadratic_form = C00 * (dx * dx) + (C01 + C10) * dx * dy + C11 * (dy * dy)
    
    # Step 5: Compute exponent
    exponent = -0.5 * quadratic_form
    
    # Step 6: Compute normalization constant
    normalization = 2.0 * math.pi * math.sqrt(det_cov)
    
    # Step 7: Return PDF
    return math.exp(exponent) / normalization
```

**Parameters:**
- No explicit tuning parameters. The shape is defined entirely by the input covariance matrix [G].

**Edge Cases and Pitfalls:**
- **Degenerate Covariance:** If $\det(\Sigma) \le 0$, the matrix is not positive-definite or is singular. The PDF is undefined or infinite. In practice, return 0.0 or clamp the determinant to a small epsilon to avoid NaNs [G].
- **Precision:** For very small determinants, $\sqrt{\det(\Sigma)}$ may underflow. For very large quadratic forms, $\exp(E)$ may underflow to 0.0. Standard floating-point arithmetic handles typical splatting scales well, but extreme aspect ratios in $\Sigma$ can cause numerical instability in the inverse calculation if not careful [G].
- **Symmetry:** The covariance matrix must be symmetric. If the input is not symmetric, the result is incorrect. The implementation assumes symmetry or uses the average of off-diagonal elements [G].

## Subtopics

### [Gradient of 2D Gaussian PDF (Conic)](3472-gradient-of-2d-gaussian-pdf-conic.md)
Computes the PDF value and its gradients with respect to the mean and covariance parameters. Choose this when training Gaussian splats and backpropagating errors to the primitive parameters.

### [2D Gaussian Probability Density (Conic)](3475-2d-gaussian-probability-density-conic.md)
Computes the unnormalized PDF using the conic matrix (inverse covariance) and a rescaling factor. Choose this when the conic representation is already available and normalization is handled separately or not required.

### [Single Point Alpha Forward](3478-single-point-alpha-forward.md)
Computes the alpha contribution of a single Gaussian splat to a pixel. Choose this when integrating the PDF into the final rasterization pipeline, including camera projection and intrinsic opacity scaling.

## Operating characteristics
- **Time Complexity:** $O(1)$ per evaluation [G].
- **Memory Complexity:** $O(1)$ auxiliary memory [G].
- **Performance:** Extremely fast, suitable for per-pixel evaluation in rasterization loops [G]. The dominant cost in a full splatting pipeline is usually the projection and sorting, not this PDF evaluation itself [G].
- **Scaling:** Scales linearly with the number of Gaussians in the rasterization loop, but each evaluation is constant time [G].
- **Strengths:** Analytically exact, numerically stable for well-conditioned covariances, no iterative approximation required [G].
- **Limitations:** Does not provide gradients (use [Gradient of 2D Gaussian PDF (Conic)](3472-gradient-of-2d-gaussian-pdf-conic.md) for that) [G]. Sensitive to degenerate covariances [G].

## Background
This method is a core component of 3D Gaussian Splatting, a technique for novel view synthesis using explicit 3D Gaussian primitives [G]. The 2D PDF evaluation is used during the rasterization phase to determine the contribution of each projected Gaussian to the final image [G]. The approach builds on the mathematical properties of Gaussian functions and their projection under affine transformations [G].

## Sources
[S1] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
