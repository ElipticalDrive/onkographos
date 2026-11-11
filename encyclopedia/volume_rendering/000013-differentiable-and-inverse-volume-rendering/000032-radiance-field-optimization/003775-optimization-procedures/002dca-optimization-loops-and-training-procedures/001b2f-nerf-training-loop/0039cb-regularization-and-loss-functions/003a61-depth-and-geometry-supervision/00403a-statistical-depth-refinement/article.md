# Statistical Depth Refinement

Also called: Corrupt Depth Patching, Underfitting Depth Smoothing

## Overview
Statistical Depth Refinement is a regularization technique used during NeRF training to prevent the model from overfitting to corrupt stereo depth maps caused by fuzzy pixels and specular highlights [S1]. It addresses the problem where direct supervision on noisy depth leads to abrupt geometric artifacts by replacing statistically identified corrupt depth values with smoother depth estimates from an underfitting model [S1]. It differs from standard depth supervision [8679-single-view-nerf-training-with-depth-map.md] by actively masking out unreliable depth pixels rather than minimizing error on all pixels, and differs from distribution-based supervision [8687-depth-supervised-nerf.md] by using a hard patching mechanism based on residual quantiles rather than soft distribution alignment.

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   $D_i$: The noisy ground-truth stereo depth map for frame $i$.
    *   $D_b^K_i$: The depth map rendered from the radiance field after $K$ iterations of training (the "underfitting" output).
    *   $\alpha$: A threshold parameter in $[0, 1]$ representing the quantile of residuals to treat as corrupt.
*   **Outputs:**
    *   $\tilde{D}_i$: The patched depth map used for subsequent gradient updates.

**Data Structures:**
*   Depth maps are 2D arrays of floating-point values, indexed by pixel coordinates $(u, v)$.
*   Residual maps are 2D arrays of non-negative floating-point values.

**Steps:**
1.  **Obtain Underfitting Depth:** Train the radiance field for $K$ iterations. At iteration $K$, render the depth map $D_b^K_i$ for the current frame $i$. This map is smoother because the model has not yet converged to fit high-frequency noise [S1].
2.  **Compute Residuals:** Calculate the absolute difference between the underfitting depth and the noisy ground-truth depth for each pixel $(u, v)$:
    $$ r_{u,v} = |D_b^K_i[u, v] - D_i[u, v]| $$
    This produces a residual map $R$ of the same dimensions as the depth maps [S1].
3.  **Compute Probabilistic Distribution:** Compute the cumulative distribution function (CDF) or histogram of the residual values in $R$ across all pixels [S1].
4.  **Identify Corrupt Pixels:** Determine the threshold value $T$ corresponding to the $(1-\alpha)$-quantile of the residual distribution. Pixels where $r_{u,v} > T$ are identified as having large residuals and are statistically classified as corrupt depth [S1].
5.  **Patch Depth:** Replace the depth values at the identified corrupt pixel locations in the ground-truth map $D_i$ with the corresponding values from the underfitting map $D_b^K_i$. The resulting map is $\tilde{D}_i$:
    $$ \tilde{D}_i[u, v] = \begin{cases} D_b^K_i[u, v] & \text{if } r_{u,v} > T \\ D_i[u, v] & \text{otherwise} \end{cases} $$
6.  **Optimize:** Use $\tilde{D}_i$ for depth loss calculation in subsequent training iterations [S1].

**Parameters:**
*   $K$: The number of training iterations before performing the refinement. Typical value: 4000 [S1].
*   $\alpha$: The fraction of pixels with the largest residuals to patch. Typical value: 0.1 (top 10%) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def statistical_depth_refinement(D_noisy, D_underfitting, alpha):
    """
    D_noisy: 2D array, noisy ground-truth depth map.
    D_underfitting: 2D array, depth map from underfitting model.
    alpha: float, quantile threshold (e.g., 0.1 for top 10%).
    Returns: patched depth map.
    """
    # Step 2: Compute residual map
    residual_map = abs(D_underfitting - D_noisy)
    
    # Step 3 & 4: Find threshold based on alpha-quantile
    # Flatten residuals to find the threshold value
    flat_residuals = residual_map.flatten()
    # Sort to find the (1-alpha) quantile value
    # We want the top alpha fraction, so we look at the value 
    # below which (1-alpha) of the data falls.
    threshold_index = int(len(flat_residuals) * (1 - alpha))
    sorted_residuals = np.sort(flat_residuals)
    threshold_value = sorted_residuals[threshold_index]
    
    # Step 5: Patch the depth
    # Create a mask for pixels with residual > threshold
    corrupt_mask = residual_map > threshold_value
    
    # Replace noisy depth with underfitting depth where mask is true
    D_patched = np.where(corrupt_mask, D_underfitting, D_noisy)
    
    return D_patched
```

**Edge Cases and Pitfalls:**
*   **Precision:** The residual calculation must use sufficient precision to distinguish between minor noise and significant corruption [S1].
*   **Boundary Conditions:** If $\alpha$ is too high, valid depth details may be smoothed out; if too low, corrupt pixels may remain [S1].
*   **Degenerate Input:** If the underfitting model produces depth values that are significantly biased (not just noisy), the patching may introduce systematic errors. The assumption is that the underfitting model averages colors and occupancy, smoothing out local noise [S1].

## Subtopics

### Perceptual Distillation
A technique that transfers deblurred texture information from pre-deblurred images to the NeRF using perceptual loss on rendered patches, overcoming geometric inconsistency of pre-deblurred images [S2]. It uses an off-the-shelf deep learning-based image deblurring network $\Theta_D$ to generate pre-deblurred images $\bar{I}$ from blurred inputs $I$ via $\bar{I} = \Theta_D(I)$ [S2]. It renders color patches $\hat{C}_{ptc}$ from the NeRF using patch-wise sampled rays $r_{ptc}^{pd}$ at the same pixel locations as the pre-deblurred color patches $\bar{C}_{ptc}$ [S2]. Perceptual loss is applied by extracting features from both the pre-deblurred color patch and the rendered color patch using a shared pre-trained image feature extractor $\phi$ [S2]. A key pitfall is that pre-deblurred images cannot be used as direct pixel-wise color supervision due to lack of 3D consistency caused by the ill-posed property of image deblurring and independent processing across multi-view images [S2].

## Operating characteristics
*   **Time Complexity:** Adds a one-time cost per refinement step (every $K$ iterations) to compute residuals and patching, which is $O(N)$ where $N$ is the number of pixels. This is negligible compared to the training iterations.
*   **Memory Complexity:** Requires storing the residual map and the underfitting depth map temporarily, $O(N)$ memory.
*   **Measured Performance:** Evaluated on robotic surgery stereo videos (DaVinci prostatectomy data), 807 frames, 15 fps [S1]. Metrics include PSNR, SSIM, and LPIPS [S1].
*   **Strengths:** Effectively prevents overfitting to corrupt depth caused by specular highlights and fuzzy pixels in surgical scenes [S1].
*   **Limitations:** Relies on the assumption that the underfitting model provides a "smoother" but geometrically plausible baseline. Requires careful tuning of $K$ and $\alpha$ [S1].
*   **When to use:** When stereo depth maps are noisy due to specularities or motion blur, and direct depth supervision causes artifacts.
*   **When not to use:** When depth maps are accurate and high-quality, as patching may remove valid high-frequency geometric details.

## Background
Originated in the context of neural rendering for stereo 3D reconstruction of deformable tissues in robotic surgery [S1]. The method was proposed to handle corrupt stereo depth caused by fuzzy pixels and specular highlights in surgical scenes [S1]. Preliminary findings showed that early-stage models (underfitting) produce smoother results in color and depth because they tend to average learned colors and occupancy, allowing the majority normal depth to smooth out minority corrupt depth [S1]. The method was implemented with $K=4000$ iterations and $\alpha=0.1$ [S1].

## Sources
[S1] Neural Rendering for Stereo 3D Reconstruction of Deformable Tissues in Robotic Surgery, https://doi.org/10.1007/978-3-031-16449-1_41, fetched 2026-10-26
[S2] Sparse-DeRF: Deblurred Neural Radiance Fields From Sparse View, https://doi.org/10.1109/tpami.2025.3563398, fetched 2026-10-27
