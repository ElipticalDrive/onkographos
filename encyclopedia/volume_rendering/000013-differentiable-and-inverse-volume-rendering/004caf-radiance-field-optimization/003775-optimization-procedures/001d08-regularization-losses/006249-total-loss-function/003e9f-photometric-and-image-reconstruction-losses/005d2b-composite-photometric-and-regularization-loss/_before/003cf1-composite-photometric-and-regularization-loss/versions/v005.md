# Composite Photometric and Regularization Loss

Also called: Combined Loss Function, Model Optimization with Loss Functions, Optimization Loss, Training Losses

## Overview
This method defines the total objective function for optimizing radiance fields by balancing photometric reconstruction accuracy against various geometric and structural regularization terms. It solves the problem of degenerate geometry (e.g., collapsed primitives, excessive thinness) and noise in learned representations by enforcing constraints on opacity, distortion, normals, and spatial smoothness. It differs from its sibling [Composite Image Loss with Visibility Weighting](4262-composite-image-loss-with-visibility-weighting.md) by explicitly incorporating primitive-level geometric regularizations (opacity, distortion, normal consistency, size) and dynamic scene constraints (flow smoothness, background entropy) rather than relying solely on visibility-weighted photometric and perceptual losses.

## How it works
The method computes a scalar loss $\mathcal{L}$ as a weighted sum of a photometric term and multiple regularization terms. The specific composition depends on the rendering primitive type (volumetric grids vs. splats) and the scene nature (static vs. dynamic).

### Inputs and Outputs
*   **Inputs:**
    *   Rendered image color $\mathbf{C}_{\text{rendered}}$ (or $\mathbf{C}_{\text{Inp}}$, $\mathbf{C}_{\text{Up}}$).
    *   Ground-truth image color $\mathbf{C}_{\text{gt}}$.
    *   Primitive parameters (e.g., voxel densities $\sigma$, Gaussian scales $\mathbf{s}$, opacities $\alpha$, vertices $\mathbf{v}$).
    *   Auxiliary fields (e.g., rendered depth $\mathbf{D}$, flow field $\mathbf{V}_{\text{flow}}$, canonical density $\mathbf{V}_{\sigma}^{\text{Can}}$).
*   **Outputs:**
    *   Scalar loss value $\mathcal{L}$ for backpropagation.

### Data Structures
The method operates on standard tensor structures provided by the differentiable renderer:
*   $\mathbf{C} \in \mathbb{R}^{H \times W \times 3}$: Rendered and ground-truth images.
*   $\mathcal{R}$: Set of sampled rays.
*   $\mathbf{V}_{\sigma}, \mathbf{V}_{\text{flow}}$: 3D voxel grids storing density and flow values.
*   $\mathbf{s}_i \in \mathbb{R}^3$: Scaling vector for the $i$-th primitive (e.g., Gaussian).
*   $\mathbf{v}_i \in \mathbb{R}^3$: Vertex positions for primitive geometry.

### Formulas and Steps
The total loss is generally structured as:
$$
\mathcal{L} = \mathcal{L}_{\text{photo}} + \sum_{k} w_k \mathcal{L}_{\text{reg}, k}
$$
where $w_k$ are hyperparameters. Two primary formulations are documented in the sources.

#### Formulation 1: Dynamic Volumetric/Grid-Based Optimization [S1]
Used for dynamic scenes with inpainted/up-sampled voxel grids.

1.  **Photometric Loss ($\mathcal{L}_{\text{photo}}$):**
    Computes the Mean Squared Error (MSE) between rendered colors from two grid states and the ground truth:
    $$
    \mathcal{L}_{\text{photo}} = \frac{1}{|\mathcal{R}|} \sum_{r \in \mathcal{R}} \left\| \mathbf{C}_{\text{Inp}}(\mathbf{r}) - \mathbf{C}_{\text{gt}}(\mathbf{r}) \right\|_2^2 + \frac{1}{|\mathcal{R}|} \sum_{r \in \mathcal{R}} \left\| \mathbf{C}_{\text{Up}}(\mathbf{r}) - \mathbf{C}_{\text{gt}}(\mathbf{r}) \right\|_2^2
    $$
    where $\mathbf{C}_{\text{Inp}}$ is rendered from an inpainted grid and $\mathbf{C}_{\text{Up}}$ from an up-sampled grid [S1].

2.  **Point Color Supervision ($\mathcal{L}_{\text{ptc}}$):**
    Directly supervises the color of sampled points, weighted by their contribution to the rendered color (following DVGO [44]) [S1].

3.  **Background Entropy Loss ($\mathcal{L}_{\text{bg}}$):**
    Encourages densities to concentrate on either foreground or background, reducing transparency artifacts [S1].

4.  **Inpaint Network Loss ($\mathcal{L}_{\text{vdiff}}$):**
    Minimizes the difference between input and output of the inpaint network:
    $$
    \mathcal{L}_{\text{vdiff}} = \left\| \mathbf{V}_{\text{R}_{\text{Inp}}}^{t} - \mathbf{V}_{\text{R}_{\text{Up}}}^{t} \right\|_1
    $$
    [S1].

5.  **Flow Regularization ($\mathcal{L}_{\text{flow}}$):**
    Encourages 3D points in canonical space to be static by computing the L1 norm of the flow field $\mathbf{V}_{\text{flow}}^{t}$ [S1].

6.  **Total Variation (TV) Regularization:**
    Applies a total variation function $\mathcal{L}_{\text{tv}}$ to enforce spatial smoothness on:
    *   Canonical density $\mathbf{V}_{\sigma}^{\text{Can}}$ [S1].
    *   Flow field $\mathbf{V}_{\text{flow}}^{t}$ [S1].
    *   Rendered depths $\mathbf{D}$ [S1].

7.  **Total Loss:**
    $$
    \mathcal{L} = \mathcal{L}_{\text{photo}} + w_1\mathcal{L}_{\text{ptc}} + w_2\mathcal{L}_{\text{bg}} + w_3\mathcal{L}_{\text{flow}} + w_4\mathcal{L}_{\text{vdiff}} + w_5\mathcal{L}_{\text{tv}}(\mathbf{V}_{\sigma}^{\text{Can}}) + w_6\mathcal{L}_{\text{tv}}(\mathbf{V}_{\text{flow}}^{t}) + w_7\mathcal{L}_{\text{tv}}(\mathbf{D})
    $$
    Weights $w_1 \sim w_7$ balance the components [S1].

#### Formulation 2: Primitive-Based (Triangle Splat) Optimization [S2]
Used for primitive-based volumetric methods like Triangle Splatting.

1.  **Photometric L1 Loss ($\mathcal{L}_1$):**
    Standard L1 loss between rendered and ground-truth images [S2].

2.  **LD-SSIM Loss ($\mathcal{L}_{\text{D-SSIM}}$):**
    A differentiable Structural Similarity Index term [S2].

3.  **Opacity Loss ($\mathcal{L}_0$):**
    Regularizes opacity values [S2].

4.  **Distortion Loss ($\mathcal{L}_d$):**
    Regularizes geometric distortion [S2].

5.  **Normal Loss ($\mathcal{L}_n$):**
    Enforces normal consistency [S2].

6.  **Size Regularization ($\mathcal{L}_s$):**
    Encourages larger triangles by penalizing small cross-product magnitudes of vertex differences. For three vertices $\{v_1, v_2, v_3\}$ sampled from a unit sphere, scaled by $d$, and centered at $q$ ($v_i = q + k \cdot d \cdot u_i$):
    $$
    \mathcal{L}_s = 2 \| (v_1 - v_0) \times (v_2 - v_0) \|^{-1}
    $$
    [S2].

7.  **Total Loss:**
    $$
    \mathcal{L} = (1 - \lambda)\mathcal{L}_1 + \lambda\mathcal{L}_{\text{D-SSIM}} + \beta_1\mathcal{L}_0 + \beta_2\mathcal{L}_d + \beta_3\mathcal{L}_n + \beta_4\mathcal{L}_s
    $$
    where $\lambda$ weights the SSIM term, and $\beta_1 \sim \beta_4$ weight the regularization terms [S2].

#### Formulation 3: Gaussian Splatting with Scaling Regularization [S3, unofficial]
Used for dynamic avatar/splatting optimization.

1.  **Photometric Loss:**
    Sum of L1 loss and Perceptual Loss ($\mathcal{L}_{\text{LPIPS}}$) [S3, unofficial].

2.  **Scaling Regularization ($\mathcal{L}_{\text{scaling}}$):**
    Prevents Gaussians from growing long and thin. Let $\hat{s}_i$ be the maximum scaling and $\tilde{s}_i$ be the minimum scaling of Gaussian $i$. Let $T_s$ be the length threshold and $T_r$ be the thinness ratio threshold.
    $$
    \mathcal{L}_{\text{scaling}}(i) = \begin{cases} |\hat{s}_i|, & \hat{s}_i > \max(T_s, T_r \tilde{s}_i) \\ 0, & \text{otherwise} \end{cases}
    $$
    [S3, unofficial].

3.  **Total Loss:**
    $$
    \mathcal{L} = \mathcal{L}_1 + \lambda_l \mathcal{L}_{\text{LPIPS}} + \lambda_s \mathcal{L}_{\text{scaling}}
    $$
    [S3, unofficial].

### Parameters
*   **Weights ($w_i, \beta_i, \lambda$):** Hyperparameters balancing the terms. Specific values are often found in supplementary materials or ablation studies [S2][S3, unofficial].
*   **Thresholds ($T_s, T_r$):** Used in scaling regularization to define "long" and "thin" primitives [S3, unofficial].
*   **Sampling:** Vertices for size regularization are sampled uniformly at random from a unit sphere, scaled, and centered [S2].

### Edge Cases and Pitfalls
*   **Degenerate Geometry:** Without size/normal/distortion regularization, primitives may collapse or become infinitely thin, leading to artifacts [S2][S3, unofficial].
*   **Transparency Artifacts:** Lack of background entropy or opacity regularization can result in semi-transparent regions where none should exist [S1].
*   **Dynamic Noise:** In dynamic scenes, lack of flow smoothness or canonical density TV regularization leads to flickering and noisy motion fields [S1].
*   **Silhouette Cues:** In monocular setups, random background mixing is required to provide silhouette cues for the optimizer [S3, unofficial].

## Subtopics

### Scaling and Background Mixing Regularization
A variant specifically for Gaussian Splatting in monocular dynamic scenes. It introduces a scaling regularization term to prevent Gaussians from becoming excessively long and thin, and uses random background color mixing to improve silhouette learning. It differs from the general primitive regularization by focusing on the aspect ratio of Gaussian scales rather than triangle geometry. Choose this when optimizing 3D Gaussians from monocular video where aspect ratio control is critical [S3, unofficial].

## Operating characteristics
*   **Complexity:** The photometric terms dominate computational cost, scaling linearly with the number of rays and pixels. Regularization terms (TV, opacity, distortion) add overhead proportional to the number of primitives or voxels.
*   **Performance:** Balancing photometric and regularization terms is critical; too much regularization degrades photometric fidelity, while too little leads to geometric artifacts [S2][S3, unofficial].
*   **Strengths:** Produces geometrically plausible and visually stable results, especially in dynamic or sparse-view scenarios [S1, S2][S3, unofficial].
*   **Limitations:** Requires careful tuning of multiple hyperparameters ($w_i, \beta_i, \lambda$) [S2][S3, unofficial].

## Background
The concept of combining photometric loss with regularization for radiance field optimization has evolved alongside differentiable rendering. Early methods focused primarily on L1/L2 photometric loss [S1]. Subsequent works introduced structural similarity (SSIM) [S2] and perceptual losses (LPIPS) [S3, unofficial] to improve visual quality. Geometric regularizations (opacity, distortion, normal consistency) were introduced to stabilize primitive-based methods like 3D Gaussian Splatting and Triangle Splatting [S2][S3, unofficial]. Dynamic scene optimization added flow and temporal smoothness constraints [S1].

## Sources
[S1] Forward Flow for Novel View Synthesis of Dynamic Scenes, https://doi.org/10.1109/iccv51070.2023.01468, fetched 2026-10-26
[S2] Triangle Splatting for Real-Time Radiance Field Rendering, https://doi.org/10.1109/3DV69130.2026.00123, fetched 2026-10-27
[S3] SplattingAvatar, https://arxiv.org/abs/2403.05087, fetched 2026-10-27 (unofficial)
