# Composite Photometric and Regularization Loss

Also called: Combined Loss Function, Model Optimization with Loss Functions, Optimization Loss, Training Losses

## Overview
This method defines the total objective function for optimizing differentiable volume rendering or splatting representations by balancing photometric reconstruction accuracy against various regularization terms. It solves the ill-posed nature of inverse rendering by enforcing structural similarity, geometric smoothness, opacity sparsity, and primitive validity. It differs from its sibling [Composite Image Loss with Visibility Weighting](4262-composite-image-loss-with-visibility-weighting.md) by explicitly incorporating non-photometric constraints such as distortion, normal consistency, and size regularization, rather than relying solely on visibility-weighted photometric errors.

## How it works
The method combines multiple loss components into a single scalar objective. The specific formulation varies depending on the representation (implicit grid, explicit primitives, or dynamic scenes).

### Inputs and Outputs
*   **Inputs:** Rendered image $\mathbf{C}$, ground truth image $\mathbf{C}_{\text{gt}}$, rendered depth $\mathbf{D}$, canonical density grid $\mathcal{V}_{\sigma}^{\text{Can}}$, motion field $\mathcal{V}_{\text{flow}}$, primitive parameters (e.g., vertices $\mathbf{v}$, scaling $\hat{\mathbf{s}}$), and ray set $\mathcal{R}$.
*   **Outputs:** Scalar loss value $\mathcal{L}$ used for gradient-based optimization.

### Data Structures
*   **Ray Batch $\mathcal{R}$:** Set of rays sampled for the current optimization step.
*   **Rendered Color $\mathbf{C}(\mathbf{r})$:** RGB color value for ray $\mathbf{r}$.
*   **Ground Truth $\mathbf{C}_{\text{gt}}(\mathbf{r})$:** RGB color value from the input image.
*   **Primitive Vertices $\mathbf{v}_i$:** 3D coordinates of primitive vertices (e.g., triangle splats).
*   **Scaling $\hat{\mathbf{s}}_i$:** Vector of scaling factors for a primitive $i$.

### Formulas and Steps
The total loss is a weighted sum of photometric and regularization terms. Two primary formulations are documented in the sources.

#### Formulation 1: Dynamic Scene Grid Optimization [S1]
Used for dynamic volumetric representations with inpainting networks.

$$
\mathcal{L} = \mathcal{L}_{\text{photo}} + w_1\mathcal{L}_{\text{ptc}} + w_2\mathcal{L}_{\text{bg}} + w_3\mathcal{L}_{\text{flow}} + w_4\mathcal{L}_{\text{vdiff}} + w_5\mathcal{L}_{\text{tv}}(\mathcal{V}_{\sigma}^{\text{Can}}) + w_6\mathcal{L}_{\text{tv}}(\mathcal{V}_{\text{flow}}^t) + w_7\mathcal{L}_{\text{tv}}(\mathbf{D})
$$

Where:
*   $\mathcal{L}_{\text{photo}}$: Photometric loss.
    $$
    \mathcal{L}_{\text{photo}} = \frac{1}{|\mathcal{R}|} \sum_{r\in \mathcal{R}} \left\| \mathbf{C}_{\text{Inp}}(\mathbf{r}) - \mathbf{C}_{\text{gt}}(\mathbf{r}) \right\|_2^2 + \frac{1}{|\mathcal{R}|} \sum_{r\in \mathcal{R}} \left\| \mathbf{C}_{\text{Up}}(\mathbf{r}) - \mathbf{C}_{\text{gt}}(\mathbf{r}) \right\|_2^2
    $$
    $\mathbf{C}_{\text{Inp}}$ is rendered from an inpainted grid; $\mathbf{C}_{\text{Up}}$ is rendered from an up-sampled grid without inpainting.
*   $\mathcal{L}_{\text{ptc}}$: Point color loss, directly supervising the color of sampled points [S1].
*   $\mathcal{L}_{\text{bg}}$: Background entropy loss, encouraging densities to concentrate on foreground or background [S1].
*   $\mathcal{L}_{\text{flow}}$: L1 norm of the motion field $\mathcal{V}_{\text{flow}}^t$, encouraging static points in canonical space [S1].
*   $\mathcal{L}_{\text{vdiff}}$: Inpaint network loss, L1 norm of the difference between input and output of the inpaint network: $\left\| \mathcal{V}_{\text{R}_{\text{Inp}}}^t - \mathcal{V}_{\text{R}_{\text{Up}}}^t \right\|_1$ [S1].
*   $\mathcal{L}_{\text{tv}}(\cdot)$: Total variation function for spatial smoothness [S1]. Applied to canonical density $\mathcal{V}_{\sigma}^{\text{Can}}$, motion $\mathcal{V}_{\text{flow}}^t$, and rendered depths $\mathbf{D}$.
*   $w_1 \sim w_7$: Weights to balance components [S1].

#### Formulation 2: Triangle Splatting [S2]
Used for explicit triangle primitive splatting.

$$
\mathcal{L} = (1 - \lambda)\mathcal{L}_1 + \lambda\mathcal{L}_{\text{D-SSIM}} + \beta_1\mathcal{L}_0 + \beta_2\mathcal{L}_d + \beta_3\mathcal{L}_n + \beta_4\mathcal{L}_s
$$

Where:
*   $\mathcal{L}_1$: Photometric L1 loss.
*   $\mathcal{L}_{\text{D-SSIM}}$: LD-SSIM (Structural Similarity Index) loss [S2].
*   $\mathcal{L}_0$: Opacity loss [S2].
*   $\mathcal{L}_d$: Distortion loss [S2].
*   $\mathcal{L}_n$: Normal consistency loss [S2].
*   $\mathcal{L}_s$: Size regularization term.
    $$
    \mathcal{L}_s = 2 \left\| (\mathbf{v}_1 - \mathbf{v}_0) \times (\mathbf{v}_2 - \mathbf{v}_0) \right\|^{-1}
    $$
    $\mathbf{v}_0, \mathbf{v}_1, \mathbf{v}_2$ are vertices of a triangle. This term encourages larger triangles by penalizing small cross-product magnitudes [S2].
*   $\lambda$: Weight for the LD-SSIM term [S2].
*   $\beta_1, \beta_2, \beta_3, \beta_4$: Weights for opacity, distortion, normal, and size regularization respectively [S2].

#### Formulation 3: Splatting Avatar Scaling Regularization [S3, unofficial]
Used for Gaussian splatting avatars to prevent degenerate shapes.

$$
\mathcal{L} = \mathcal{L}_1 + \lambda_l \mathcal{L}_{\text{LPIPS}} + \lambda_s \mathcal{L}_{\text{scaling}}
$$

Where:
*   $\mathcal{L}_{\text{scaling}}(i)$: Scaling regularization for primitive $i$.
    $$
    \mathcal{L}_{\text{scaling}}(i) = \begin{cases} |\hat{s}_i|, & \hat{s}_i > \max(T_s, T_r \tilde{s}_i) \\ 0, & \text{otherwise} \end{cases}
    $$
    $\hat{s}_i$ is the maximum scaling value, $\tilde{s}_i$ is the minimum scaling value, $T_s$ is the length threshold, and $T_r$ is the thinness ratio threshold [S3, unofficial].
*   $\mathcal{L}_{\text{LPIPS}}$: Perceptual loss (Learned Perceptual Image Patch Similarity) [S3, unofficial].
*   $\lambda_l, \lambda_s$: Weights for LPIPS and scaling regularization [S3, unofficial].

### Parameters
*   **Weights ($w_i, \beta_i, \lambda$):** Hyperparameters balancing the terms. Specific values are often found in supplementary materials or ablation studies [S2][S3, unofficial].
*   **Thresholds ($T_s, T_r$):** Used in scaling regularization to define "long" and "thin" primitives [S3, unofficial].
*   **Batch Size ($|\mathcal{R}|$):** Number of rays sampled per iteration.

### Edge Cases and Pitfalls
*   **Degenerate Primitives:** Without size regularization (e.g., $\mathcal{L}_s$ or $\mathcal{L}_{\text{scaling}}$), primitives may collapse into lines or points, causing numerical instability in depth or normal calculations [S2][S3, unofficial].
*   **Background Ambiguity:** In dynamic scenes, background entropy loss $\mathcal{L}_{\text{bg}}$ helps disambiguate foreground/background, but requires careful weighting to avoid suppressing valid background details [S1].
*   **Precision:** The inverse norm in $\mathcal{L}_s$ can explode if the triangle area approaches zero. Implementation should clamp the cross-product magnitude or handle degenerate triangles explicitly [G].

## Subtopics

### Scaling and Background Mixing Regularization
A variant used in Gaussian splatting for monocular video (e.g., SplattingAvatar). It introduces a random background color mixing strategy to provide silhouette cues and a specific scaling regularization term to prevent Gaussians from growing long and thin. It differs from standard photometric loss by explicitly penalizing aspect ratio anomalies in primitive scaling [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of rays $|\mathcal{R}|$ and primitives. The total variation and SSIM terms add overhead proportional to image resolution.
*   **Memory Complexity:** Depends on the representation. Grid-based methods require storing canonical density and motion fields [S1]. Primitive-based methods require storing vertex and scaling parameters [S2][S3, unofficial].
*   **Strengths:** Improves geometric consistency and visual quality by preventing artifacts like floating geometry, blurry normals, or degenerate primitives.
*   **Limitations:** Requires careful tuning of multiple hyperparameters ($w_i, \beta_i, \lambda$). Over-regularization can lead to oversmoothed results or loss of fine details [S2][S3, unofficial].
*   **When to Use:** When optimizing differentiable radiance fields or splats from sparse or monocular inputs where geometric priors are needed to resolve ambiguities.

## Background
The concept of combining photometric and regularization losses is fundamental to differentiable rendering. The specific formulations cited originate from recent advances in dynamic scene reconstruction [S1], triangle-based splatting [S2], and Gaussian splatting for avatars [S3, unofficial]. The use of LD-SSIM and distortion losses is common in modern radiance field optimization [S2].

## Sources
[S1] Forward Flow for Novel View Synthesis of Dynamic Scenes, https://doi.org/10.1109/iccv51070.2023.01468, fetched 2026-10-26
[S2] Triangle Splatting for Real-Time Radiance Field Rendering, https://doi.org/10.1109/3DV69130.2026.00123, fetched 2026-10-27
[S3] SplattingAvatar, https://arxiv.org/abs/2403.05087, fetched 2026-10-27 (unofficial)
