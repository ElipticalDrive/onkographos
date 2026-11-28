# Color-Corrected Error Metrics

Also called: Per-image polynomial color correction, Photometric variation mitigation metrics

## Overview
Color-Corrected Error Metrics are evaluation procedures designed to mitigate the impact of photometric variations—such as autoexposure shifts, white balance changes, and RGB clipping—in "in the wild" datasets [S1]. They solve the problem of ill-posed view synthesis evaluation where the ground truth image's photometric conditions may not match the model's neutral rendering [S1]. Unlike standard metrics that penalize brightness or color shifts as errors, these metrics first align the rendered image to the ground truth via a per-image least-squares fit, making models like Stable View Synthesis and GLO quantitatively comparable [S1].

## How it works
**Inputs:**
- Rendered image $\hat{I} \in \mathbb{R}^{H \times W \times 3}$ [S1]
- Ground truth image $I \in \mathbb{R}^{H \times W \times 3}$ [S1]
- Saturated pixel mask $M \in \{0, 1\}^{H \times W \times 3}$ indicating pixels to ignore [S1]

**Outputs:**
- Corrected error metric value (e.g., PSNR, SSIM, LPIPS) computed on the aligned images [S1]

**Data Structures:**
- Standard image buffers and a small coefficient vector for the polynomial fit [G]

**Steps:**
1. **Identify Saturated Pixels:** Determine the set of pixels in the ground truth image $I$ that are clipped or saturated. These pixels are excluded from the fitting process [S1].
2. **Fit Polynomial Expansion:** Solve a per-image least-squares problem to find coefficients for a quadratic polynomial expansion that maps the rendered RGB values $\hat{I}$ to the ground truth $I$ [S1].
3. **Apply Correction:** Transform the rendered image using the fitted polynomial: $I_{\text{corr}} = f(\hat{I})$ [S1].
4. **Compute Metric:** Calculate the desired error metric between $I_{\text{corr}}$ and $I$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def color_corrected_error(rendered_img, gt_img, saturated_mask):
    """
    rendered_img: np.array, shape (H, W, 3), float32
    gt_img: np.array, shape (H, W, 3), float32
    saturated_mask: np.array, shape (H, W, 3), bool (True if pixel is saturated/clipped)
    """
    # 1. Filter out saturated pixels
    valid_pixels = ~saturated_mask
    if not np.any(valid_pixels):
        return np.inf  # No valid pixels to fit

    # 2. Solve least squares for quadratic polynomial expansion
    # Quadratic expansion in 3 channels: 1, r, g, b, r^2, g^2, b^2, rg, rb, gb (10 terms)
    r, g, b = rendered_img[valid_pixels][:, 0], rendered_img[valid_pixels][:, 1], rendered_img[valid_pixels][:, 2]
    X = np.column_stack([
        np.ones(len(r)), r, g, b,
        r**2, g**2, b**2, r*g, r*b, g*b
    ])
    y = gt_img[valid_pixels]  # Shape (N, 3)
    # Solve for coefficients beta (shape: 10, 3)
    beta, _, _, _ = np.linalg.lstsq(X, y, rcond=None)

    # 3. Apply the fitted transformation to the entire rendered image
    r_all, g_all, b_all = rendered_img[:, :, 0], rendered_img[:, :, 1], rendered_img[:, :, 2]
    X_all = np.column_stack([
        np.ones_like(r_all), r_all, g_all, b_all,
        r_all**2, g_all**2, b_all**2, r_all*g_all, r_all*b_all, g_all*b_all
    ])
    I_corr = X_all @ beta

    # 4. Compute standard metric (e.g., PSNR) between I_corr and gt_img
    mse = np.mean((I_corr - gt_img) ** 2)
    psnr = 10 * np.log10(1.0 / mse) if mse > 0 else np.inf
    return psnr
```

**Parameters:**
- Polynomial degree: Quadratic (2nd order) [S1]
- Saturated pixel threshold: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Saturated Pixels:** Ignoring saturated pixels is critical; including them would bias the polynomial fit towards the clipped values, which do not contain valid scene information [S1].
- **Ill-posedness:** Without this correction, metrics become ill-posed because the model cannot be expected to replicate arbitrary autoexposure or white balance settings [S1].

## Subtopics

### [Textural Complexity Objective](13002-textural-complexity-objective.md)
The Textural Complexity Objective ($T$) is a metric used in active view planning for radiance-field optimization to identify camera poses that capture regions with high photometric variation. Unlike geometric metrics that analyze 3D mesh structure, $T$ operates on 2D image projections, leveraging the fact that textural characteristics are readily discernible in 2D even if 3D geometry is ambiguous. It serves as a component of Color-Corrected Error Metrics, helping to mitigate photometric variations such as autoexposure shifts or white balance changes in "in the wild" datasets by prioritizing views with rich texture.

### Bradford Chromatic Adaptation for White Balance
A post-processing white balance technique using Bradford chromatic adaptation to map from a Planck/CIE-daylight illuminant to D65 [S2, unofficial]. It applies the Bradford chromatic adaptation transform, differing from simple gray-world or white-patch algorithms [S2, unofficial].

### Phys3DGS Relighting Evaluation
For synthetic data, explicitly computes a 3-channel rescale factor for base color by taking the median value of all pixel-wise ratios between the model's prediction and ground truth [S3, unofficial]. For real captures, omits the rescale procedure and presents only qualitative results as no ground truth on base color is provided [S3, unofficial]. The PBR rendering pipeline operates in linear (HDR) space, and the final result is clipped to [0, 1] and mapped to SRGB just before calculating photometric loss or evaluation [S3, unofficial].

### Color perturbation augmentation
A data augmentation technique simulating variable lighting and exposure by applying a random scale and shift transformation to RGB pixel values across training images [S4]. For each training image $I_i \in [0, 1]^{800 \times 800 \times 3}$, scale $s_{ij}$ and offset $b_{ij}$ are sampled uniformly. The transformation is applied to each channel $j$: $\tilde{I}_{ij} = \min(1, \max(0, s_{ij} I_{ij} + b_{ij}))$ where $s_{ij} \sim \mathcal{U}(0.8, 1.2)$ and $b_{ij} \sim \mathcal{U}(-0.2, 0.2)$ [S4]. Perturbations are applied to all training images except the first, which serves as the reference view [S4]. On the Lego dataset, this perturbation alone decreases NeRF's PSNR by approximately 10 dB on average compared to the original unperturbed dataset [S4].

### Camera Localization Evaluation
Verification of camera localization performance on NeRF synthetic dataset scenes using reprojection error versus photometric error [S5, unofficial]. Using reprojection error improves performance more than using photometric error alone, consistent with iNeRF results [S5, unofficial]. In scenes with insufficient color uniqueness (e.g., Materials), reprojection error may degrade performance due to mismatches between corresponding points [S5, unofficial]. In scenes with many local solutions for photometric errors (e.g., Mic, Ship), reprojection error does not avoid local solutions and does not improve performance [S5, unofficial]. A potential improvement is to propagate unique features other than color to empty regions in the same way as color fields [S5, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- The fitting step involves solving a least-squares problem per image. The complexity depends on the number of polynomial terms (quadratic implies ~10 terms for RGB) and the number of valid pixels [G]. It is generally fast relative to the rendering process itself [G].
- Memory usage is minimal, requiring only storage for the image buffers and the small set of polynomial coefficients [G].

**Measured Performance:**
- On the Tanks and Temples dataset, using these color-corrected metrics allows for quantitative comparison between models like Stable View Synthesis (SVS) and models with Global Lighting Optimization (GLO), which would otherwise be incomparable due to photometric shifts [S1].
- The correction makes SVS and the GLO model variant quantitatively comparable [S1].

**Strengths and Limitations:**
- **Strengths:** Mitigates the impact of autoexposure and white balance variations in "in the wild" datasets [S1].
- **Limitations:** Does not solve the underlying issue of photometric variation during training; it is an evaluation procedure [S1]. Requires identification of saturated pixels, which may be ambiguous in some datasets [S1].

**When to Use:**
- Use when evaluating view synthesis on datasets with uncontrolled lighting or camera settings (e.g., Tanks and Temples, Phototourism) [S1, S4].
- Use when comparing models that handle photometric variation differently (e.g., explicit appearance embeddings vs. neutral rendering) [S1].

**When Not to Use:**
- Do not use for synthetic datasets with controlled, known lighting conditions where photometric consistency is expected and required [S1].
- Do not use if the goal is to evaluate the model's ability to replicate specific lighting conditions (e.g., relighting tasks) [S3, unofficial].

## Background
Originating from the challenges of evaluating neural radiance fields on "in the wild" datasets, this method addresses the issue where camera autoexposure and white balance vary between images, making standard error metrics misleading [S1]. The approach was highlighted in the context of the Mip-NeRF 360 paper, which constructed a dataset with fixed photometric conditions to avoid these issues, but acknowledged the need for such metrics for existing uncontrolled datasets like Tanks and Temples [S1]. The concept of mitigating photometric variation is also related to NeRF in the Wild (NeRF-W), which uses appearance embeddings to isolate tinting, but Color-Corrected Error Metrics specifically refer to the post-hoc evaluation correction [S4].

## Sources
[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S2] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
[S3] Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2409.10335, fetched 2026-10-26 (unofficial)
[S4] NeRF in the Wild: Neural Radiance Fields for Unconstrained Photo Collections, https://doi.org/10.1109/cvpr46437.2021.00713, fetched 2026-10-26
[S5] Neural Density-Distance Fields, https://doi.org/10.48550/arXiv.2207.14455, fetched 2026-10-27 (unofficial)
