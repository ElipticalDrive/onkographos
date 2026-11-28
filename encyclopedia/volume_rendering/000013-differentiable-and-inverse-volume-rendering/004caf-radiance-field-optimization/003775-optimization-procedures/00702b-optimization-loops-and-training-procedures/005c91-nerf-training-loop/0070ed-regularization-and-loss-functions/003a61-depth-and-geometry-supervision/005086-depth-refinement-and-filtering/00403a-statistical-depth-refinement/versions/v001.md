# Statistical Depth Refinement

Also called: Corrupt Depth Patching, Underfitting Depth Smoothing

## Overview
Statistical Depth Refinement is a post-processing step applied during the training of radiance fields to mitigate the impact of corrupt stereo depth data caused by fuzzy pixels and specular highlights [S1]. It prevents the model from overfitting to these noisy depth values, which typically result in abrupt geometric artifacts [S1]. Unlike standard depth supervision which treats all depth pixels equally, this method identifies outliers based on the residual between the current model's output and the noisy stereo depth, replacing them with smoother values from an underfitting model state [S1].

## How it works
**Inputs:**
*   $D_i$: The noisy stereo depth map for the $i$-th frame [S1].
*   $D_b^K_i$: The depth map rendered by the radiance field after $K$ iterations of training (the "underfitting" state) [S1].
*   $\alpha$: A threshold parameter in $[0, 1]$ representing the quantile of residuals to treat as corrupt [S1].
*   $K$: The iteration count at which the underfitting depth is sampled [S1].

**Outputs:**
*   $\tilde{D}_i$: The refined depth map where corrupt pixels are replaced by smoother values [S1].

**Data Structures:**
*   Depth maps are 2D arrays of floating-point values corresponding to pixel coordinates $(u, v)$ [S1].
*   Residual maps are 2D arrays of absolute differences [S1].

**Steps:**
1.  **Sample Underfitting Depth:** After $K$ training iterations, render the depth map $D_b^K_i$ for the current frame [S1].
2.  **Compute Residuals:** Calculate the absolute difference between the underfitting depth and the noisy stereo depth for each pixel $(u, v)$:
    $$ r_{u,v} = |D_b^K_i[u, v] - D_i[u, v]| $$
    This produces a residual map $R$ [S1].
3.  **Determine Threshold:** Compute the $\alpha$-quantile of the values in $R$. Let $T_\alpha$ be the value such that a fraction $\alpha$ of the residuals are greater than or equal to $T_\alpha$ [S1].
4.  **Identify Corrupt Pixels:** Create a binary mask $M$ where $M_{u,v} = 1$ if $r_{u,v} \ge T_\alpha$, and $0$ otherwise. Pixels with $M_{u,v}=1$ are identified as corrupt [S1].
5.  **Patch Depth:** Generate the refined depth map $\tilde{D}_i$ by replacing the values in $D_i$ at corrupt locations with the corresponding values from the underfitting depth $D_b^K_i$:
    $$ \tilde{D}_i[u, v] = \begin{cases} D_b^K_i[u, v] & \text{if } M_{u,v} = 1 \\ D_i[u, v] & \text{otherwise} \end{cases} $$
    [S1]
6.  **Optimize:** Use $\tilde{D}_i$ for depth supervision in subsequent training iterations [S1].

**Parameters:**
*   $K = 4000$: The number of iterations after which the underfitting depth is sampled [S1].
*   $\alpha = 0.1$: The quantile threshold for identifying corrupt depth [S1].

**Edge Cases and Pitfalls:**
*   If the underfitting model has not yet converged to a reasonable shape, the "smooth" values may be geometrically incorrect, potentially smoothing out valid fine details [S1].
*   The method assumes that the majority of depth pixels are correct and that corrupt pixels are statistical outliers [S1].

## Subtopics

### Perceptual Distillation
A technique that transfers deblurred texture information from pre-deblurred images to the NeRF using perceptual loss on rendered patches, overcoming the geometric inconsistency inherent in pre-deblurred images [S2]. It uses an off-the-shelf deep learning-based image deblurring network $\Theta_D$ to generate pre-deblurred images $\bar{I}$ from blurred inputs $I$ via $\bar{I} = \Theta_D(I)$ [S2]. Rendered color patches $\hat{C}_{ptc}$ are generated from patch-wise sampled rays $r_{ptc}^{pd}$ at the same pixel locations as the pre-deblurred color patches $\bar{C}_{ptc}$ [S2]. A shared pre-trained image feature extractor $\phi$ extracts features from both patches, and a perceptual loss is applied [S2]. This is chosen when texture fidelity is critical but direct pixel-wise supervision is unreliable due to 3D inconsistency [S2]. A key pitfall is that pre-deblurred images cannot be used as direct pixel-wise color supervision because the ill-posed nature of deblurring and independent processing across multi-view images leads to incoherent results [S2].

## Operating characteristics
*   **Time Complexity:** Adds a one-time cost per refinement step (every $K$ iterations) to compute residuals, sort/quantile, and patch. This is negligible compared to the main training loop [S1].
*   **Memory Complexity:** Requires storing the residual map and the mask, which is $O(N)$ where $N$ is the number of pixels [S1].
*   **Strengths:** Effectively removes artifacts caused by specular highlights and fuzzy pixels in stereo depth [S1].
*   **Limitations:** Relies on the assumption that early training outputs are smoother and more robust to noise than the noisy input [S1].
*   **When to use:** When using noisy stereo depth maps (e.g., from low-quality sensors or challenging scenes like surgical tools) [S1].
*   **When not to use:** When high-precision, clean depth maps are available, or when the early training state is not representative of the final geometry [S1].

## Background
Statistical Depth Refinement was proposed in the context of neural rendering for stereo 3D reconstruction of deformable tissues in robotic surgery [S1]. The authors observed that standard depth supervision overfits to corrupt depth data caused by fuzzy pixels and specular highlights, leading to abrupt artifacts [S1]. The method leverages the observation that underfitting models tend to average learned colors and occupancy, resulting in smoother depth maps that can serve as a robust prior for identifying outliers [S1]. It is applied to DaVinci robotic prostatectomy data [S1].

## Sources
[S1] Neural Rendering for Stereo 3D Reconstruction of Deformable Tissues in Robotic Surgery, https://doi.org/10.1007/978-3-031-16449-1_41, fetched 2026-10-26
[S2] Sparse-DeRF: Deblurred Neural Radiance Fields From Sparse View, https://doi.org/10.1109/tpami.2025.3563398, fetched 2026-10-27
