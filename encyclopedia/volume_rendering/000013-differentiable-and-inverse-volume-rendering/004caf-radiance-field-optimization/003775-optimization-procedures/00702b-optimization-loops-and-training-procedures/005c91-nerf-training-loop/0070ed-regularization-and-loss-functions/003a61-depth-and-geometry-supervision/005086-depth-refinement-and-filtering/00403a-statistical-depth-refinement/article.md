# Statistical Depth Refinement

Also called: Corrupt Depth Patching, Underfitting Depth Smoothing

## Overview
Statistical Depth Refinement is a post-processing step applied during the training of radiance fields to mitigate artifacts caused by corrupt stereo depth inputs, such as those arising from fuzzy pixels or specular highlights in surgical scenes [S1]. It prevents the model from overfitting to these noisy depth values by replacing them with smoother depth estimates from an underfitting model state [S1]. Unlike direct depth supervision which may lock in errors, this method statistically identifies outliers in the residual between the current model's depth and the noisy input, replacing them with the "averaged" output of the early-stage model [S1].

## How it works
**Inputs:**
*   $D_i$: The noisy stereo depth map for the $i$-th frame [S1].
*   $D_b^K_i$: The depth map rendered by the radiance field after $K$ iterations of training [S1].
*   $\alpha$: A threshold parameter in $[0, 1]$ defining the quantile of largest residuals to treat as corrupt [S1].

**Outputs:**
*   $\widehat{D}_i$: The patched depth map used for subsequent optimization [S1].

**Data Structures:**
*   Residual Map $R$: A 2D array of the same spatial dimensions as $D_i$ and $D_b^K_i$, where each element $R[u,v]$ stores the absolute difference between the depth values at pixel $(u,v)$.

**Steps:**
1.  **Render Underfitting Depth:** After $K$ iterations of training, render the depth map $D_b^K_i$ for the current frame $i$. At this early stage, the model tends to average learned colors and occupancy, producing smoother depth estimates [S1].
2.  **Compute Residuals:** Calculate the absolute difference between the underfitting depth and the noisy stereo depth to form the residual map $R$:
    $$ R[u,v] = | D_b^K_i[u,v] - D_i[u,v] | $$
    where $(u,v)$ is the pixel coordinate [S1].
3.  **Compute Probabilistic Distribution:** Compute the cumulative distribution function (CDF) or rank order of the values in $R$ [S1].
4.  **Identify Corrupt Pixels:** Determine the threshold value corresponding to the $(1-\alpha)$-quantile of the residual distribution. Identify all pixels $(u,v)$ where $R[u,v]$ exceeds this threshold. These pixels are statistically identified as having large residuals and are classified as corrupt depth [S1].
5.  **Patch Depth:** Replace the depth values of the identified corrupt pixels in the noisy depth map $D_i$ with the corresponding values from the smoother underfitting depth map $D_b^K_i$. The resulting map is $\widehat{D}_i$ [S1].
6.  **Optimize:** Use the patched depth map $\widehat{D}_i$ for depth supervision in subsequent training iterations [S1].

**Parameters:**
*   $K$: The number of training iterations after which the underfitting depth is sampled. Typical value: 4000 [S1].
*   $\alpha$: The fraction of pixels with the largest residuals to patch. Typical value: 0.1 [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def statistical_depth_refinement(noisy_depth_D_i, model, K, alpha):
    """
    Refines noisy stereo depth by patching outliers with underfitting model output.
    
    Args:
        noisy_depth_D_i: 2D array, noisy stereo depth map for current frame.
        model: Radiance field model.
        K: Number of training iterations to wait for underfitting state.
        alpha: Quantile threshold for identifying corrupt pixels (e.g., 0.1).
        
    Returns:
        patched_depth: 2D array, depth map with corrupt pixels replaced.
    """
    # Step 1: Obtain underfitting depth after K iterations
    # Note: In practice, this requires pausing optimization or maintaining a snapshot
    # of the model state at iteration K.
    underfitting_depth_D_b = model.render_depth(iteration=K)
    
    # Step 2: Compute residual map
    residual_map_R = abs(underfitting_depth_D_b - noisy_depth_D_i)
    
    # Step 3 & 4: Identify pixels with largest residuals (top alpha quantile)
    # Flatten residuals to find the threshold value
    flat_residuals = residual_map_R.flatten()
    threshold_value = np.percentile(flat_residuals, 100 * (1 - alpha))
    
    # Create mask for corrupt pixels
    corrupt_mask = residual_map_R > threshold_value
    
    # Step 5: Patch corrupt pixels
    # Replace noisy depth with underfitting depth where mask is True
    patched_depth = np.where(corrupt_mask, underfitting_depth_D_b, noisy_depth_D_i)
    
    return patched_depth
```

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on the assumption that the underfitting model is smoother than the noisy input. If the model has already overfit by iteration $K$, the "smooth" reference may also contain artifacts [S1].
*   **Boundary Conditions:** The residual calculation is per-pixel. Discontinuities at object boundaries may be treated as corrupt if the stereo depth is noisy there, potentially leading to over-smoothing of valid geometry [S1].
*   **Degenerate Input:** If the stereo depth is uniformly corrupt or the model fails to converge, the statistical distribution may not separate "normal" from "corrupt" effectively [S1].

## Subtopics

### Perceptual Distillation
A technique that transfers deblurred texture information from pre-deblurred images to the NeRF using perceptual loss on rendered patches, overcoming the geometric inconsistency of pre-deblurred images [S2]. It uses an off-the-shelf deep learning-based image deblurring network to generate pre-deblurred images from blurred inputs [S2]. It renders color patches $\hat{C}_{ptc}$ from the NeRF using patch-wise sampled rays $r_{ptc}^{pd}$ at the same pixel locations as the pre-deblurred color patches $\bar{C}_{ptc}$ [S2]. Perceptual loss is applied by extracting features from both the pre-deblurred color patch and the rendered color patch using a shared pre-trained image feature extractor [S2]. Pre-deblurred images cannot be used as direct pixel-wise color supervision due to lack of 3D consistency caused by the ill-posed property of image deblurring and independent processing across multi-view images [S2]. [S2]

### Depth Refinement
A local, per-pixel geometry supervision method that generates refined depth candidates via spatial neighbors and multiplicative perturbations, scored by normalized cross-correlation (NCC) [S3, unofficial]. For training camera $C$ and pixel $p$, it forms a candidate set $D(p) = \{ \hat{D}_t(C)(p), \hat{D}_t(C)(p+e_x), \hat{D}_t(C)(p+e_y) \} \cup \{ \hat{D}_t(C)(p)(1+\epsilon_k) \mid k=1..n_{rand} \}$, where $\epsilon_k \sim \text{Unif}(-2s, 2s)$, $s \ll 1$ [S3, unofficial]. Each candidate is backprojected from $C$ and reprojected into $K$ nearest views $N(C)$ [S3, unofficial]. Each candidate is scored by NCC between fixed-size, bilinear-sampled patches [S3, unofficial]. The candidate with the highest score is selected as refined depth $D_t^*(p)$ [S3, unofficial]. Refined depths are rejected if reprojection error exceeds a small threshold [S3, unofficial]. [S3, unofficial]

## Operating characteristics
*   **Time Complexity:** The method adds a one-time cost per refinement step (every $K$ iterations) proportional to the number of pixels $N$ in the depth map, primarily for residual calculation and sorting/quantile determination $O(N \log N)$ or $O(N)$ with selection algorithms [S1].
*   **Memory Complexity:** Requires storage for the residual map and the mask, $O(N)$ additional memory [S1].
*   **Measured Performance:** Evaluated on robotic surgery stereo videos (DaVinci robotic prostatectomy data), 6 cases, 807 frames, 15 fps [S1]. Metrics include PSNR, SSIM, and LPIPS [S1].
*   **Strengths:** Effectively handles corrupt stereo depth caused by fuzzy pixels and specular highlights [S1]. Prevents overfitting to corrupt depth which leads to abrupt artifacts [S1].
*   **Limitations:** Relies on the availability of stereo depth inputs [S1]. Requires careful tuning of $K$ and $\alpha$ [S1].
*   **When to use:** When training radiance fields with noisy stereo depth supervision, particularly in scenes with specular highlights or fuzzy pixels [S1].
*   **When not to use:** When high-quality, clean depth supervision is available, or when computational resources are extremely limited and the overhead of maintaining an underfitting snapshot is prohibitive [S1].

## Background
Statistical Depth Refinement was proposed in the context of neural rendering for stereo 3D reconstruction of deformable tissues in robotic surgery [S1]. The method addresses the challenge of using stereo depth from surgical cameras, which often suffer from noise due to specular highlights and fuzzy pixels [S1]. The authors observed that early-stage (underfitting) radiance fields produce smoother depth estimates, which can be used to identify and patch corrupt depth values in the noisy stereo input [S1]. The method was implemented with $K=4000$ iterations and $\alpha=0.1$ [S1].

## Sources
[S1] Neural Rendering for Stereo 3D Reconstruction of Deformable Tissues in Robotic Surgery, https://doi.org/10.1007/978-3-031-16449-1_41, fetched 2026-10-26
[S2] Sparse-DeRF: Deblurred Neural Radiance Fields From Sparse View, https://doi.org/10.1109/tpami.2025.3563398, fetched 2026-10-27
[S3] Advancing Structured Priors for Sparse-Voxel Surface Reconstruction, https://doi.org/10.48550/arxiv.2601.17720, fetched 2026-10-26 (unofficial)
