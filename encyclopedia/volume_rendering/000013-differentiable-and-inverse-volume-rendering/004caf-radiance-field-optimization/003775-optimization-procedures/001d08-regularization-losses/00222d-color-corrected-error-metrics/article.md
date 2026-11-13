# Color-Corrected Error Metrics

Also called: Per-image polynomial color correction, Quadratic RGB fitting for evaluation

## Overview
Color-Corrected Error Metrics are an evaluation procedure designed to mitigate the impact of photometric variation—such as autoexposure shifts, white balance changes, and RGB clipping—in "in the wild" datasets like Tanks and Temples [S1]. By fitting a per-image transformation to align rendered colors with ground truth before computing loss, this method renders quantitative comparisons between different view synthesis models (e.g., SVS vs. GLO) robust to camera-induced photometric inconsistencies [S1]. It sets itself apart from standard metrics by explicitly modeling non-linear color shifts rather than assuming a fixed linear relationship or ignoring photometric drift entirely.

## How it works
**Inputs and Outputs:**
- **Input:** A rendered image $\hat{I}$ (RGB values) and a ground truth image $I_{gt}$ (RGB values) for a specific camera pose.
- **Output:** A set of correction coefficients (quadratic polynomial weights) and the corrected error metric value.

**Data Structures:**
- **Pixel Data:** Two arrays of size $W \times H \times 3$ containing floating-point RGB values for $\hat{I}$ and $I_{gt}$.
- **Mask:** A boolean array of size $W \times H$ indicating valid (non-saturated) pixels.
- **Coefficients:** A vector of 12 values per color channel (or 12 total if shared, though typically per-channel for flexibility): $[c_0, c_1, c_2, c_3, c_4, c_5, c_6, c_7, c_8, c_9, c_{10}, c_{11}]$.

**Steps:**
1. **Identify Valid Pixels:** Create a mask $M$ where $M_{u,v} = 1$ if the corresponding pixel in $I_{gt}$ is not saturated (i.e., no channel value is clipped to 0 or 1, or the maximum representable value), and $0$ otherwise. Saturated pixels are ignored to prevent the least squares fit from being biased by clipped data [S1].
2. **Construct Design Matrix:** For each valid pixel $(u,v)$, construct a feature vector $\mathbf{x}_{u,v}$ representing the quadratic expansion of the rendered RGB values. For a single channel, if $\hat{r}$ is the rendered value, the expansion is typically $[1, \hat{r}, \hat{r}^2]$. For 3 channels, this expands to 10 terms (1, R, G, B, R², G², B², RG, RB, GB) or 12 if cross-terms are handled differently. The source specifies a "quadratic polynomial expansion" [S1]. Let $\mathbf{X}$ be the matrix where each row is $\mathbf{x}_{u,v}^T$ for all valid pixels.
3. **Solve Least Squares:** For each color channel $k \in \{R, G, B\}$, solve the linear least squares problem:
   $$
   \min_{\mathbf{w}_k} \| \mathbf{X} \mathbf{w}_k - \mathbf{y}_k \|_2^2
   $$
   where $\mathbf{y}_k$ is the vector of ground truth values for channel $k$ at valid pixels. The solution is:
   $$
   \mathbf{w}_k = (\mathbf{X}^T \mathbf{X})^{-1} \mathbf{X}^T \mathbf{y}_k
   $$
   [S1].
4. **Apply Correction:** Compute the corrected rendered image $\hat{I}'$ by applying the fitted polynomial to each pixel:
   $$
   \hat{I}'_{u,v,k} = \mathbf{x}_{u,v}^T \mathbf{w}_k
   $$
5. **Compute Metric:** Calculate the desired error metric (e.g., PSNR, SSIM, LPIPS) between $\hat{I}'$ and $I_{gt}$ using only the valid pixels or the full image if the metric allows [S1].

**Parameters:**
- **Polynomial Degree:** Quadratic (degree 2) [S1].
- **Saturated Pixel Threshold:** Not explicitly defined in the sources, but implied to be binary clipping detection [S1].

**Edge Cases and Pitfalls:**
- **Saturated Pixels:** If too many pixels are saturated, the least squares problem may become ill-conditioned or the correction may be unreliable [S1].
- **Overfitting:** A quadratic fit may overfit noise in low-texture regions.
- **Non-Differentiability:** This method is used for *evaluation*, not training. The correction is applied post-hoc to the rendered image, so it does not propagate gradients to the scene parameters during optimization [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def color_corrected_error(rendered_img, gt_img, metric_func):
    """
    rendered_img: (H, W, 3) float32, values in [0, 1] or HDR
    gt_img: (H, W, 3) float32, values in [0, 1] or HDR
    metric_func: function to compute error between corrected render and gt
    """
    # Step 1: Identify valid pixels (ignore saturated)
    # Assuming saturation is defined as any channel == 0 or == 1 (for [0,1] range)
    # For HDR, this threshold may vary. Source implies ignoring clipped RGB [S1].
    saturated = (rendered_img == 0.0) | (rendered_img == 1.0)
    valid_mask = ~np.any(saturated, axis=-1) # (H, W) boolean
    
    # Flatten valid pixels
    valid_indices = np.where(valid_mask)
    r_render = rendered_img[valid_indices] # (N, 3)
    g_render = rendered_img[valid_indices]
    b_render = rendered_img[valid_indices]
    
    r_gt = gt_img[valid_indices]
    g_gt = gt_img[valid_indices]
    b_gt = gt_img[valid_indices]
    
    # Step 2: Construct Design Matrix for Quadratic Expansion
    # Terms: 1, R, G, B, R^2, G^2, B^2, RG, RB, GB
    # Note: Source says "quadratic polynomial expansion" [S1]. 
    # Standard quadratic in 3 vars has 10 terms. 
    # We construct X for each channel independently or jointly? 
    # Usually per-channel fitting is done for color correction.
    
    # Build X matrix (N, 10)
    X = np.column_stack([
        np.ones(len(r_render)),
        r_render, g_render, b_render,
        r_render**2, g_render**2, b_render**2,
        r_render * g_render, r_render * b_render, g_render * b_render
    ])
    
    # Step 3: Solve Least Squares for each channel
    # Using pseudo-inverse for stability
    XtX = X.T @ X
    Xty = np.array([X.T @ r_gt, X.T @ g_gt, X.T @ b_gt])
    
    # Solve (XtX) w = Xty
    # w shape: (10, 3)
    try:
        W = np.linalg.solve(XtX, Xty)
    except np.linalg.LinAlgError:
        # Fallback to pseudo-inverse if singular
        W = np.linalg.lstsq(X, np.column_stack([r_gt, g_gt, b_gt]), rcond=None)[0]
        
    # Step 4: Apply Correction
    # Compute corrected values for all valid pixels
    corrected_r = X @ W[:, 0]
    corrected_g = X @ W[:, 1]
    corrected_b = X @ W[:, 2]
    
    corrected_img = np.zeros_like(rendered_img)
    corrected_img[valid_indices] = np.column_stack([corrected_r, corrected_g, corrected_b])
    
    # Step 5: Compute Metric
    return metric_func(corrected_img, gt_img)
```

## Subtopics

### Bradford Chromatic Adaptation for White Balance
A post-processing white balance technique that uses the Bradford chromatic adaptation transform to map colors from a Planck/CIE-daylight illuminant to the D65 standard illuminant [S2, unofficial]. Unlike simple gray-world or white-patch algorithms, this method explicitly models the chromatic adaptation matrix [S2, unofficial]. It is applied as part of an imager pipeline before tone mapping [S2, unofficial].

### Phys3DGS Relighting Evaluation
A specific evaluation protocol for physically-based rendering that addresses material-lighting ambiguity [S3, unofficial]. For synthetic data, it computes a 3-channel rescale factor for base color by taking the median of pixel-wise ratios between the model's prediction and ground truth [S3, unofficial]. For real captures, this rescale is omitted, and only qualitative results are presented due to the lack of ground truth base color [S3, unofficial]. The PBR pipeline operates in linear HDR space, clipping to [0, 1] and mapping to sRGB only before calculating photometric loss or evaluation [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ per image for the least squares fit, where $N$ is the number of valid pixels. The matrix inversion is constant time ($10 \times 10$ or $12 \times 12$).
- **Memory Complexity:** $O(N)$ to store the design matrix and intermediate vectors.
- **Strengths:** Robust to autoexposure and white balance variations in uncontrolled datasets [S1]. Allows fair comparison between models that may have different global color characteristics [S1].
- **Limitations:** Does not correct for geometric errors or lighting direction mismatches, only photometric intensity/color shifts [S1]. Relies on the assumption that a quadratic polynomial is sufficient to model the camera's response [S1].
- **When to use:** Evaluating view synthesis on "in the wild" datasets like Tanks and Temples where camera settings are not fixed [S1].
- **When not to use:** When ground truth is available with fixed, calibrated camera settings, or when evaluating geometric accuracy rather than photometric consistency.

## Background
The method was introduced in the context of evaluating neural radiance fields on the Tanks and Temples dataset, where photometric variation (autoexposure, white balance, clipping) makes standard metrics like PSNR and SSIM unreliable [S1]. It builds on the observation that "in the wild" images pose an ill-posed task for view synthesis evaluation if photometric conditions are not standardized [S1]. The approach is distinct from training-time color correction, serving purely as an evaluation metric adjustment [S1].

## Sources
[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S2] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
[S3] Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2409.10335, fetched 2026-10-26 (unofficial)
