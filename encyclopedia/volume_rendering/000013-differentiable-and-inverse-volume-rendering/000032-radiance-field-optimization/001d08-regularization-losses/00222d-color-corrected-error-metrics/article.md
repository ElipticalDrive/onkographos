# Color-Corrected Error Metrics

Also called: Per-image quadratic color correction, Photometric variation mitigation metrics

## Overview
Color-Corrected Error Metrics are an evaluation procedure designed to mitigate photometric variations—such as autoexposure, white balance shifts, and RGB clipping—in "in the wild" datasets like Tanks and Temples [S1]. By solving a per-image least squares problem to fit a quadratic polynomial expansion of rendered RGB values to ground truth, the method neutralizes global brightness and color shifts, making view synthesis models quantitatively comparable despite uncontrolled camera settings [S1].

## How it works
**Inputs:** Rendered image $I_{render}$ (RGB), Ground truth image $I_{true}$ (RGB), and a mask $M_{valid}$ identifying non-saturated pixels.
**Outputs:** Corrected error metrics (PSNR, SSIM, LPIPS) computed on the color-adjusted render.

**Data Structures:**
- $I_{render}, I_{true}$: 2D arrays of shape $(H, W, 3)$, values in $[0, 1]$.
- $M_{valid}$: Boolean mask of shape $(H, W)$, true where pixel is not saturated/clipped.

**Steps:**
1. **Mask Saturated Pixels:** Identify pixels where any channel in $I_{true}$ or $I_{render}$ is clipped (e.g., $\ge 1.0$ or $\le 0.0$). These are excluded from the fitting process [S1].
2. **Construct Design Matrix:** For each valid pixel $p$, construct a feature vector $x_p$ representing the quadratic expansion of the rendered RGB values. For a pixel with rendered color $r = [R, G, B]$, the expansion includes terms for linear components, squared components, and cross-products:
   $$ x_p = [1, R, G, B, R^2, G^2, B^2, RG, RB, GB]^T $$
   The constant term $1$ accounts for bias/offset [S1].
3. **Solve Least Squares:** Solve for the coefficient vector $c$ that minimizes the squared error between the transformed render and the ground truth for all valid pixels:
   $$ \min_c \sum_{p \in M_{valid}} \| I_{true}[p] - X_p c \|^2 $$
   where $X_p$ is the row vector $x_p^T$. This is solved via the normal equation:
   $$ c = (X^T X)^{-1} X^T Y $$
   where $X$ is the matrix of all $x_p$ and $Y$ is the vector of all $I_{true}[p]$ for valid pixels [S1].
4. **Apply Correction:** Compute the corrected rendered image $I_{corrected}$ by applying the fitted polynomial to every pixel in $I_{render}$:
   $$ I_{corrected}[p] = X_p c $$
5. **Compute Metrics:** Calculate standard error metrics (PSNR, SSIM, LPIPS) between $I_{corrected}$ and $I_{true}$ [S1].

**Parameters:**
- Polynomial degree: Quadratic (order 2) [S1].
- Saturated pixel threshold: Typically 0.0 or 1.0, depending on data normalization [S1].

**Edge Cases:**
- **No Valid Pixels:** If all pixels are saturated, the least squares problem is ill-posed. The sources do not specify a fallback, but logically, metrics cannot be computed [S1].
- **Precision:** Inverting $X^T X$ may be numerically unstable if the rendered colors are highly correlated or degenerate. Standard numerical linear algebra routines should be used [G].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def color_corrected_metrics(render, true, saturated_threshold=0.99):
    """
    render: (H, W, 3) array, values in [0, 1]
    true: (H, W, 3) array, values in [0, 1]
    saturated_threshold: float, pixels with any channel >= this are ignored
    """
    # 1. Mask saturated pixels
    # Assuming saturation is defined as values close to 1.0 or 0.0. 
    # The source mentions "ignoring saturated pixels" and "clipped RGB".
    # We define valid pixels as those not near the boundaries.
    valid_mask = (render < saturated_threshold).all(axis=-1) & \
                 (render > 1.0 - saturated_threshold).all(axis=-1) & \
                 (true < saturated_threshold).all(axis=-1) & \
                 (true > 1.0 - saturated_threshold).all(axis=-1)
    
    # Flatten valid pixels
    render_flat = render[valid_mask] # (N, 3)
    true_flat = true[valid_mask]     # (N, 3)
    
    if render_flat.shape[0] == 0:
        raise ValueError("No valid non-saturated pixels found.")

    # 2. Construct Design Matrix for quadratic expansion
    # Terms: 1, R, G, B, R^2, G^2, B^2, RG, RB, GB
    R, G, B = render_flat[:, 0], render_flat[:, 1], render_flat[:, 2]
    X = np.column_stack([
        np.ones_like(R),
        R, G, B,
        R**2, G**2, B**2,
        R*G, R*B, G*B
    ])
    
    # 3. Solve Least Squares: min || Xc - Y ||^2
    # Using numpy's least squares solver for numerical stability
    # Y is true_flat (N, 3)
    c, residuals, rank, s = np.linalg.lstsq(X, true_flat, rcond=None)
    
    # 4. Apply Correction
    render_corrected_flat = X @ c # (N, 3)
    
    # 5. Compute Metrics (Placeholder for actual metric implementation)
    # In practice, one would compute PSNR, SSIM, LPIPS between 
    # render_corrected_flat and true_flat, then reshape or average.
    # Here we just return the corrected image for metric calculation.
    return render_corrected_flat
```

## Subtopics

### Bradford Chromatic Adaptation for White Balance
A post-processing white balance technique that uses the Bradford chromatic adaptation transform to map colors from a Planck/CIE-daylight illuminant to D65 [S2, unofficial]. Unlike simple gray-world or white-patch algorithms, it explicitly models the chromatic adaptation process, providing a more physically grounded correction for white balance in rendering pipelines [S2, unofficial].

## Operating characteristics
**Time Complexity:** $O(N)$ where $N$ is the number of valid pixels, dominated by the matrix construction and the least squares solve. The least squares solve is typically $O(N \cdot D^2)$ or $O(N \cdot D^3)$ where $D=10$ is the number of polynomial terms, which is constant and small.
**Memory Complexity:** $O(N)$ to store the design matrix and intermediate vectors.
**Strengths:** Effectively neutralizes global brightness and color shifts caused by autoexposure and white balance variations in "in the wild" datasets, allowing for fair comparison between models [S1].
**Limitations:** Does not correct for local photometric variations or non-linear tone mapping curves. It assumes a global quadratic relationship between the render and the ground truth, which may not hold for all scenes or camera responses [S1].
**When to use:** Evaluating view synthesis on datasets with uncontrolled camera settings (e.g., Tanks and Temples) where autoexposure/white balance varies between images [S1].
**When not to use:** When the dataset has fixed, calibrated camera settings, or when local color accuracy is more important than global color consistency [S1].

## Background
The method was introduced in the context of evaluating Neural Radiance Fields (NeRF) on the Tanks and Temples dataset, where photometric variation (autoexposure, white balance, clipping) makes standard evaluation metrics ill-posed [S1]. It builds on earlier work by Martin-Brualla et al. who investigated specialized training and evaluation procedures for such datasets [S1]. The approach was used to compare the Mip-NeRF 360 model with Stable View Synthesis (SVS) [S1].

## Sources
[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S2] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
