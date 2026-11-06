# Color-Corrected Error Metrics


## Overview
Color-Corrected Error Metrics are an evaluation procedure designed to mitigate photometric variations—such as autoexposure, white balance shifts, and RGB clipping—in "in the wild" datasets like Tanks and Temples [S1]. By solving a per-image least squares problem to fit a quadratic polynomial expansion of the rendering's RGB values to the true image, the method neutralizes global brightness and color shifts, making view synthesis models quantitatively comparable [S1].

## How it works
**Inputs:** Rendered image $I_{render}$ (RGB), ground truth image $I_{true}$ (RGB), and a validity mask for non-saturated pixels [G].
**Outputs:** Corrected error metrics (e.g., PSNR, SSIM, LPIPS) computed on the color-adjusted render [G].

**Data Structures:**
- $I_{render}$ and $I_{true}$: 2D arrays of shape $(H, W, 3)$ with values in $[0, 1]$ [G].
- $M_{valid}$: Boolean mask of shape $(H, W)$ identifying pixels to include in the fit [G].

**Steps:**
1. **Identify Valid Pixels:** Exclude pixels where any RGB channel in $I_{render}$ or $I_{true}$ is saturated or clipped [S1].
2. **Fit Polynomial:** Solve a per-image least squares problem to find coefficients for a quadratic polynomial expansion of the rendered RGB values that best predicts the true image values for the valid pixels [S1].
3. **Apply Correction:** Transform the entire rendered image using the fitted polynomial coefficients [S1].
4. **Compute Metrics:** Evaluate standard error metrics between the corrected render and the ground truth [S1].

**Parameters:**
- Polynomial degree: Quadratic (order 2) [S1].
- Saturated pixel threshold: Not given in the sources.

**Edge Cases:**
- **No Valid Pixels:** If all pixels are saturated, the least squares problem is ill-posed. Not given in the sources.
- **Precision:** Inverting $X^T X$ may be numerically unstable if rendered colors are highly correlated. Standard numerical linear algebra routines (e.g., QR decomposition or SVD) should be used [G].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def color_corrected_metrics(render, true, saturated_threshold=0.99):
    """
    render: (H, W, 3) array, values in [0, 1]
    true: (H, W, 3) array, values in [0, 1]
    saturated_threshold: float, pixels with any channel >= this are ignored [G]
    """
    # 1. Mask saturated pixels [G]
    valid_mask = (render < saturated_threshold).all(axis=-1) & \
                 (render > 1.0 - saturated_threshold).all(axis=-1) & \
                 (true < saturated_threshold).all(axis=-1) & \
                 (true > 1.0 - saturated_threshold).all(axis=-1)

    render_flat = render[valid_mask]
    true_flat = true[valid_mask]

    if render_flat.shape[0] == 0:
        raise ValueError("No valid non-saturated pixels found.")

    # 2. Construct quadratic expansion features [G]
    R, G, B = render_flat[:, 0], render_flat[:, 1], render_flat[:, 2]
    X = np.column_stack([
        np.ones_like(R),
        R, G, B,
        R**2, G**2, B**2,
        R*G, R*B, G*B
    ])

    # 3. Solve least squares for polynomial coefficients [S1]
    c, _, _, _ = np.linalg.lstsq(X, true_flat, rcond=None)

    # 4. Apply correction to all pixels [S1]
    R_all, G_all, B_all = render[:, :, 0], render[:, :, 1], render[:, :, 2]
    X_all = np.column_stack([
        np.ones_like(R_all).flatten(),
        R_all.flatten(), G_all.flatten(), B_all.flatten(),
        R_all.flatten()**2, G_all.flatten()**2, B_all.flatten()**2,
        R_all.flatten()*G_all.flatten(), R_all.flatten()*B_all.flatten(), G_all.flatten()*B_all.flatten()
    ])
    render_corrected_flat = X_all @ c
    render_corrected = render_corrected_flat.reshape(render.shape)

    # 5. Compute metrics on corrected image [G]
    mse = np.mean((render_corrected - true) ** 2)
    psnr = 10.0 * np.log10(1.0 / mse) if mse > 0 else float('inf')
    return render_corrected, psnr
```

## Subtopics

### Bradford Chromatic Adaptation for White Balance
A post-processing white balance technique that uses the Bradford chromatic adaptation transform to map colors from a Planck/CIE-daylight illuminant to D65 [S2, unofficial]. Unlike simple gray-world or white-patch algorithms, it explicitly models the chromatic adaptation process, providing a more physically grounded correction for white balance in rendering pipelines [S2, unofficial].

## Operating characteristics
**Time Complexity:** Not given in the sources.
**Memory Complexity:** Not given in the sources.
**Strengths:** Effectively neutralizes global brightness and color shifts caused by autoexposure and white balance variations, allowing fair comparison between models [S1].
**Limitations:** Not given in the sources.
**When to use:** Evaluating view synthesis on datasets with uncontrolled camera settings where autoexposure/white balance varies between images [S1].
**When not to use:** Not given in the sources.

## Background
The method was introduced in the context of evaluating Neural Radiance Fields on the Tanks and Temples dataset, where photometric variation makes standard evaluation metrics ill-posed [S1]. It builds on earlier work by Martin-Brualla et al. who investigated specialized training and evaluation procedures for such datasets [S1]. The approach was used to compare the Mip-NeRF 360 model with Stable View Synthesis (SVS) [S1].

## Sources
[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S2] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
