# 2D Gaussian Probability Density (Standard)

Also called: 2D Gaussian PDF

## Overview
Computes the probability density function (PDF) value of a 2D Gaussian at a query point `xy`, given its mean vector and covariance matrix [S1, unofficial]. It differs from the [Conic](3475-2d-gaussian-probability-density-conic.md) variant by operating on the explicit covariance matrix rather than the pre-computed conic matrix [S1, unofficial].

## How it works
**Inputs:**
- `xy`: A 2D vector representing the query point [S1, unofficial].
- `gaussian_mean`: A 2D vector representing the mean of the Gaussian [S1, unofficial].
- `gaussian_covariance`: A $2 \times 2$ covariance matrix $\Sigma$ [S1, unofficial].

**Outputs:**
- `pdf`: A scalar float representing the probability density value at `xy` [S1, unofficial].

**Data Structures:**
- `xy`, `gaussian_mean`: 2-element arrays or vectors [S1, unofficial].
- `gaussian_covariance`: $2 \times 2$ matrix stored as four floats `[cov[0,0], cov[0,1], cov[1,0], cov[1,1]]` [S1, unofficial].

**Steps:**
1. Compute mean difference: `xy_mean = xy - gaussian_mean` [S1, unofficial].
2. Compute determinant: `det_cov = gaussian_covariance.determinant()` [S1, unofficial].
3. Compute inverse covariance matrix analytically for $2 \times 2$: `inv_cov = (1 / det_cov) * [[cov[1,1], -cov[0,1]], [-cov[1,0], cov[0,0]]]` [S1, unofficial].
4. Compute quadratic form: `xy_mean_T_cov_inv = xy_mean @ inv_cov` followed by `xy_mean_T_cov_inv_xy_mean = xy_mean_T_cov_inv @ xy_mean` [S1, unofficial].
5. Compute exponent: `exponent = -0.5 * xy_mean_T_cov_inv_xy_mean` [S1, unofficial].
6. Return PDF: `exp(exponent) / (2 * pi * sqrt(det_cov))` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_2d_gaussian_pdf_standard(xy, gaussian_mean, gaussian_covariance):
    # Step 1: Compute mean difference [S1, unofficial]
    xy_mean = [xy[0] - gaussian_mean[0], xy[1] - gaussian_mean[1]]

    # Step 2: Compute determinant [S1, unofficial]
    cov_00 = gaussian_covariance[0, 0]
    cov_01 = gaussian_covariance[0, 1]
    cov_10 = gaussian_covariance[1, 0]
    cov_11 = gaussian_covariance[1, 1]
    det_cov = cov_00 * cov_11 - cov_01 * cov_10

    # Step 3: Compute inverse covariance matrix analytically [S1, unofficial]
    inv_00 = cov_11 / det_cov
    inv_01 = -cov_01 / det_cov
    inv_10 = -cov_10 / det_cov
    inv_11 = cov_00 / det_cov

    # Step 4: Compute quadratic form [S1, unofficial]
    xy_mean_T_cov_inv_x = xy_mean[0] * inv_00 + xy_mean[1] * inv_10
    xy_mean_T_cov_inv_y = xy_mean[0] * inv_01 + xy_mean[1] * inv_11
    xy_mean_T_cov_inv_xy_mean = xy_mean_T_cov_inv_x * xy_mean[0] + xy_mean_T_cov_inv_y * xy_mean[1]

    # Step 5: Compute exponent [S1, unofficial]
    exponent = -0.5 * xy_mean_T_cov_inv_xy_mean

    # Step 6: Return PDF [S1, unofficial]
    pdf = math.exp(exponent) / (2.0 * math.pi * math.sqrt(det_cov))
    return pdf
```

**Parameters:** Not given in the sources.

**Edge Cases and Pitfalls:** Not given in the sources.

## Subtopics

### [Gradient of 2D Gaussian PDF (Conic)](3472-gradient-of-2d-gaussian-pdf-conic.md)
Computes the PDF value and its gradients with respect to the mean and covariance for optimization. It provides normalized, unnormalized, and conic-based variants. Choose this when training Gaussian splats and gradients are needed.

### [2D Gaussian Probability Density (Conic)](3475-2d-gaussian-probability-density-conic.md)
Computes the unnormalized PDF using the conic matrix (inverse covariance) and a rescaling factor. Choose this for performance-critical rendering where the covariance is already in conic form.

### [Single Point Alpha Forward](3478-single-point-alpha-forward.md)
Computes the alpha value for a single 3D Gaussian projected to 2D. It builds on the Standard PDF by handling the full projection pipeline. Choose this for rasterizing a single Gaussian.

## Operating characteristics

## Background
The implementation details are derived from the Taichi-based 3D Gaussian Splatting repository [S1, unofficial].

## Sources
[S1] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
