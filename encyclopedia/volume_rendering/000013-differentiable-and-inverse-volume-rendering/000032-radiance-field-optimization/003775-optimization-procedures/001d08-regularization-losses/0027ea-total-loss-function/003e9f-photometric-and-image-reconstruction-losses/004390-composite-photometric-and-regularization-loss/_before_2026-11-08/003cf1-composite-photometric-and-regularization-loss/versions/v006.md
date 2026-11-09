# Composite Photometric and Regularization Loss

Also called: Combined Loss Function, Model Optimization with Loss Functions, Optimization Loss, Training Losses

## Overview
This method defines the total objective function for optimizing radiance fields by balancing photometric reconstruction accuracy against multiple regularization terms that constrain geometric and appearance properties. It addresses the ill-posed nature of inverse volume rendering by preventing degeneracies such as transparency artifacts, geometric distortion, and excessive primitive scaling. It differs from its sibling [Composite Image Loss with Visibility Weighting](4262-composite-image-loss-with-visibility-weighting.md) by explicitly incorporating structural similarity (SSIM), opacity concentration, distortion, normal consistency, and specific primitive-scale penalties, rather than relying solely on visibility-weighted photometric errors.

## How it works
The method computes a scalar loss $\mathcal{L}$ as a weighted sum of a primary photometric term and one or more regularization terms. The specific formulation varies depending on the rendering primitive type (volumetric grids vs. splats) and scene nature (static vs. dynamic).

### Inputs and Outputs
*   **Inputs:** Rendered image $\mathbf{C}_{\text{rendered}}$ (or $\mathbf{C}_{\text{Inp}}, \mathbf{C}_{\text{Up}}$), ground-truth image $\mathbf{C}_{\text{gt}}$, sampled rays $\mathcal{R}$, primitive parameters (e.g., density $\sigma$, flow $V^{flow}$, scaling $\mathbf{s}$, vertices $\mathbf{v}$), and depth maps $\mathbf{D}$.
*   **Outputs:** Scalar loss value $\mathcal{L}$ and gradients for optimization.

### Data Structures
*   **Rays $\mathcal{R}$:** Set of rays sampled for the current batch.
*   **Volumetric Grids:** Fields for density $\mathbf{V}_{\sigma}^{Can}$, flow $\mathbf{V}_{flow}^{t}$, and color/features.
*   **Splats/Primitives:** Parameters including scaling $\mathbf{s}_i$, position $\mathbf{q}$, and vertices $\mathbf{v}_i$.

### Formulas and Steps

#### 1. Photometric Loss ($\mathcal{L}_{\text{photo}}$)
The photometric term measures the difference between rendered and ground-truth colors. Two common variants exist:

**Variant A: Multi-grid MSE (Dynamic Scenes)**
Used when inpainted and up-sampled grids are rendered [S1].
$$
\mathcal{L}_{\text{photo}} = \frac{1}{|\mathcal{R}|} \sum_{r \in \mathcal{R}} \left\| \mathbf{C}_{\text{Inp}}(\mathbf{r}) - \mathbf{C}_{\text{gt}}(\mathbf{r}) \right\|_2^2 + \frac{1}{|\mathcal{R}|} \sum_{r \in \mathcal{R}} \left\| \mathbf{C}_{\text{Up}}(\mathbf{r}) - \mathbf{C}_{\text{gt}}(\mathbf{r}) \right\|_2^2
$$
where $\mathbf{C}_{\text{Inp}}$ is rendered from an inpainted grid and $\mathbf{C}_{\text{Up}}$ from an up-sampled grid without inpainting [S1].

**Variant B: L1 + Perceptual (Splats)**
Used for Gaussian splatting with background mixing [S3, unofficial].
$$
\mathcal{L}_{\text{photo}} = \mathcal{L}_1 + \lambda_l \mathcal{L}_{\text{LPIPS}}
$$
where $\mathcal{L}_1$ is the L1 norm and $\mathcal{L}_{\text{LPIPS}}$ is the perceptual loss [S3, unofficial].

#### 2. Structural Similarity Loss ($\mathcal{L}_{\text{D-SSIM}}$)
In some formulations, a differentiable SSIM term is blended with L1 [S2]:
$$
\mathcal{L}_{\text{struct}} = (1 - \lambda)\mathcal{L}_1 + \lambda\mathcal{L}_{\text{D-SSIM}}
$$
where $\lambda$ is the weight for the LD-SSIM term [S2].

#### 3. Regularization Terms
*   **Opacity Loss ($\mathcal{L}_0$):** Encourages densities to concentrate on foreground or background, reducing semi-transparent regions [S2].
*   **Distortion Loss ($\mathcal{L}_d$):** Penalizes geometric distortion [S2].
*   **Normal Consistency Loss ($\mathcal{L}_n$):** Enforces normal consistency [S2].
*   **Point Color Supervision ($\mathcal{L}_{ptc}$):** Directly supervises the color of sampled points, weighted by their contribution to the rendered color [S1].
*   **Background Entropy Loss ($\mathcal{L}_{bg}$):** Encourages densities to concentrate on either foreground or background [S1].
*   **Flow Loss ($\mathcal{L}_{flow}$):** The L1 norm of the flow field $\mathbf{V}_{flow}^{t}$ in canonical space, encouraging most 3D points to be static [S1].
*   **Total Variation (TV) Loss ($\mathcal{L}_{tv}$):** Enforces spatial smoothness on density, flow, and depth [S1].
    $$
    \mathcal{L}_{tv}(\mathbf{X}) = \text{TotalVariation}(\mathbf{X})
    $$
    Applied to $\mathbf{V}_{\sigma}^{Can}$, $\mathbf{V}_{flow}^{t}$, and rendered depths $\mathbf{D}$ [S1].
*   **Inpainting Difference Loss ($\mathcal{L}_{vdiff}$):** L1 norm of the difference between inpainted and up-sampled voxel grids [S1].
    $$
    \mathcal{L}_{vdiff} = \left\| \mathbf{V}_{\text{R}_{\text{Inp}}}^{t} - \mathbf{V}_{\text{R}_{\text{Up}}}^{t} \right\|_1
    $$
*   **Size Regularization ($\mathcal{L}_s$):** Encourages larger triangles in triangle splatting by penalizing small cross-product magnitudes of vertex differences [S2].
    $$
    \mathcal{L}_s = 2 \left\| (\mathbf{v}_1 - \mathbf{v}_0) \times (\mathbf{v}_2 - \mathbf{v}_0) \right\|^{-1}
    $$
    where $\mathbf{v}_0, \mathbf{v}_1, \mathbf{v}_2$ are vertices of a triangle [S2].
*   **Scaling Regularization ($\mathcal{L}_{scaling}$):** Prevents Gaussians from growing long and thin [S3, unofficial].
    $$
    \mathcal{L}_{scaling}(i) = \begin{cases} |\hat{s}_i|, & \hat{s}_i > \max(T_s, T_r \tilde{s}_i) \\ 0, & \text{otherwise} \end{cases}
    $$
    where $\hat{s}_i$ is the max scaling, $\tilde{s}_i$ is the min scaling, $T_s$ is the length threshold, and $T_r$ is the thinness ratio threshold [S3, unofficial].

#### 4. Total Loss Combinations

**Combination A: Volumetric/Dynamic (Grid-based)**
$$
\mathcal{L} = \mathcal{L}_{\text{photo}} + w_1\mathcal{L}_{ptc} + w_2\mathcal{L}_{bg} + w_3\mathcal{L}_{flow} + w_4\mathcal{L}_{vdiff} + w_5\mathcal{L}_{tv}(\mathbf{V}_{\sigma}^{Can}) + w_6\mathcal{L}_{tv}(\mathbf{V}_{flow}^{t}) + w_7\mathcal{L}_{tv}(\mathbf{D})
$$
where $w_1 \sim w_7$ are weights to balance components [S1].

**Combination B: Triangle Splatting**
$$
\mathcal{L} = (1 - \lambda)\mathcal{L}_1 + \lambda\mathcal{L}_{\text{D-SSIM}} + \beta_1\mathcal{L}_0 + \beta_2\mathcal{L}_d + \beta_3\mathcal{L}_n + \beta_4\mathcal{L}_s
$$
where $\beta_1 \sim \beta_4$ are weights for opacity, distortion, normal, and size regularization respectively [S2].

**Combination C: Gaussian Splatting with Background Mixing**
$$
\mathcal{L} = \mathcal{L}_1 + \lambda_l \mathcal{L}_{\text{LPIPS}} + \lambda_s \mathcal{L}_{scaling}
$$
This variant uses a random background color generated every iteration to mix with $\mathbf{I}$ and $\mathbf{I}_{gt}$ to provide silhouette cues [S3, unofficial].

### Parameters
*   **Weights ($w_k, \beta_k, \lambda$):** Hyperparameters balancing components. Specific values are often detailed in supplementary materials [S2].
*   **Thresholds ($T_s, T_r$):** Used in scaling regularization [S3, unofficial].
*   **Perceptual Weight ($\lambda_l$):** Weight for LPIPS loss [S3, unofficial].
*   **SSIM Weight ($\lambda$):** Weight for D-SSIM term [S2].

### Edge Cases and Pitfalls
*   **Degenerate Primitives:** Without size/regularization, primitives may collapse or become infinitely thin [S2][S3, unofficial].
*   **Transparency Artifacts:** Without opacity/entropy loss, semi-transparent regions may persist [S1, S2].
*   **Flickering:** Without TV regularization on flow/depth, dynamic scenes may exhibit noise [S1].
*   **Silhouette Ambiguity:** In monocular settings, background mixing is required to provide silhouette cues [S3, unofficial].

## Subtopics

### Scaling and Background Mixing Regularization
A variant specifically for Gaussian splatting in monocular settings. It prevents Gaussians from growing long and thin by applying a conditional penalty on the maximum scaling $\hat{s}_i$ when it exceeds a length threshold $T_s$ and a thinness ratio $T_r$ times the minimum scaling $\tilde{s}_i$. It also employs background mixing, where a random background color is generated every iteration and mixed with the rendered and ground-truth images to provide silhouette cues [S3, unofficial].

## Operating characteristics
*   **Complexity:** The photometric term scales linearly with the number of rays $|\mathcal{R}|$. Regularization terms (TV, opacity, distortion) scale with the number of primitives or grid voxels.
*   **Performance:** Balancing multiple terms requires careful tuning of weights ($w_k, \beta_k$) to avoid one term dominating the gradient [S1, S2].
*   **Strengths:** Prevents common failure modes in inverse rendering (degeneracy, transparency, flickering) [S1, S2][S3, unofficial].
*   **Limitations:** Increased computational overhead due to multiple loss components. Requires careful hyperparameter tuning [S2].

## Background
The concept of combining photometric and regularization losses is central to differentiable volume rendering and radiance field optimization. Early works like DVGO used point color supervision [S1]. Later methods introduced opacity regularization [S2], distortion loss [S2], and normal consistency [S2]. Dynamic scene rendering introduced flow and TV regularization [S1]. Gaussian splatting variants added scaling regularization and background mixing for monocular inputs [S3, unofficial].

## Sources
[S1] Forward Flow for Novel View Synthesis of Dynamic Scenes, https://doi.org/10.1109/iccv51070.2023.01468, fetched 2026-10-26
[S2] Triangle Splatting for Real-Time Radiance Field Rendering, https://doi.org/10.1109/3DV69130.2026.00123, fetched 2026-10-27
[S3] SplattingAvatar, https://arxiv.org/abs/2403.05087, fetched 2026-10-27 (unofficial)
