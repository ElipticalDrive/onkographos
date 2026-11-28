# HeadGaS change all

Also called: Ours change all, HeadGaS change all parameters

## Overview
"HeadGaS change all" is an ablation variant of the HeadGaS method that modifies the parameter prediction strategy by allowing all Gaussian attributes to be dynamically adjusted via blending, rather than relying on a fixed basis with only positional and rotational shifts. It differs from the primary HeadGaS method [Parent](17304-gaussian-head-avatar.md) by using a learned feature basis to predict color, opacity, position shifts, and rotation quaternations directly, whereas the primary method typically uses a more constrained deformation model. It serves as a baseline to verify the efficacy of the full blending mechanism compared to simpler conditioning or no blending.

## How it works
**Inputs and Outputs:**
*   **Input:** Blended feature vector $\boldsymbol{f}_i$ for each Gaussian $i$.
*   **Output:** Spherical Harmonic (SH) coefficients for color, opacity scalar, 3 values for position shift $\Delta \boldsymbol{\mu}$, and 4 values for rotation represented as a quaternion $\boldsymbol{q}$.

**Data Structures:**
*   **Blended Feature $\boldsymbol{f}_i$:** A latent vector derived from the expression parameters and encoded position, passed through the learned feature basis $F$ of HeadGaS. The exact dimensionality of $\boldsymbol{f}_i$ is not specified in the sources, but it serves as the conditioning input to the MLP.
*   **MLP $\phi''(\cdot)$:** A multi-layer perceptron that maps $\boldsymbol{f}_i$ to the output parameters. The architecture is not fully detailed, but it outputs the specific fields listed above.

**Steps:**
1.  **Feature Blending:** Compute the blended feature $\boldsymbol{f}_i$ using the learned feature basis $F$ of HeadGaS. This basis allows the expression parameters to act as weights for blending Gaussian features [S1].
2.  **Parameter Prediction:** Feed $\boldsymbol{f}_i$ into the MLP $\phi''(\cdot)$. The MLP outputs:
    *   Spherical Harmonic coefficients for color.
    *   Opacity value.
    *   3 values representing the position shift $\Delta \boldsymbol{\mu}$.
    *   4 values representing the rotation as a quaternion $\boldsymbol{q}$.
3.  **Transformation Application:** Apply the position shift and rotation transformation to the base Gaussian parameters in the same manner as described in the "Ours w/ $\Delta(\boldsymbol{\mu}, \boldsymbol{R})$" variant [S1]. This involves updating the position $\boldsymbol{\mu}$ and rotation $\boldsymbol{R}$ (derived from $\boldsymbol{q}$) of the Gaussian.
4.  **Regularization:** Apply the regularization term in the loss function as defined in equation (10) of the source [S1]. The specific formula for eq. (10) is not provided in the notes, but it is applied consistently with the primary method.

**Formulas:**
*   **Output Prediction:**
    $$ [\text{Color}_{SH}, \alpha, \Delta \boldsymbol{\mu}, \boldsymbol{q}] = \phi''(\boldsymbol{f}_i) $$
    Where:
    *   $\text{Color}_{SH}$: Spherical Harmonic coefficients.
    *   $\alpha$: Opacity.
    *   $\Delta \boldsymbol{\mu}$: Position shift vector (3 values).
    *   $\boldsymbol{q}$: Rotation quaternion (4 values).
*   **Transformation:** The position and rotation are updated using $\Delta \boldsymbol{\mu}$ and $\boldsymbol{q}$ as described in the parent method's transformation logic [S1].

**Parameters:**
*   **MLP Architecture:** Not explicitly defined for $\phi''(\cdot)$ in the notes, but it is distinct from the 5-layer MLP used in the "w/o blending" variant [S1].
*   **Feature Basis $F$:** Learned during training, used to blend features based on expression parameters [S1].

**Edge Cases and Pitfalls:**
*   **Quaternion Normalization:** Quaternions must be normalized to ensure valid rotations. This is a standard requirement for quaternion-based rotations [G].
*   **Blending Stability:** The blending mechanism relies on the learned feature basis $F$. Degenerate inputs or out-of-distribution expression parameters could lead to unstable feature blends [G].

## Subtopics

### HeadGaS Ablation (No Perceptual Loss)
An ablation variant of HeadGaS where the perceptual loss component is disabled from the total loss function. The total loss is defined as $\mathcal {L}_\text {total} = \lambda _{1} \mathcal {L}_{1}(I_\text {r}, I_\text {gt}) + \lambda _{s} \mathcal {L}_\text {SSIM} (I_\text {r}, I_\text {gt})$, where $\lambda_1$ and $\lambda_s$ are weights, $\mathcal{L}_1$ is L1 loss, $\mathcal{L}_{SSIM}$ is SSIM loss, $I_r$ is the rendered image, and $I_{gt}$ is the ground truth image [S1]. On the INSTA dataset with 234k Gaussians, it achieves a rendering time of 0.019s and a PSNR of 29.83 [S1].

### HeadGaS w/o blending
An ablation variant that verifies using expression parameters as a weight for blending Gaussian features works better than using it as a simple condition to the MLP. It does not use a basis of latent features $F$. Instead, the expression vector $\boldsymbol{e}_i$ and the encoded position $\boldsymbol{\mu}$ are fed directly into an MLP $\phi(\cdot)$ to predict color and opacity. The MLP consists of five linear layers, each followed by a leaky ReLU, extended to reach a performance plateau [S1]. On the INSTA dataset with 135k Gaussians, it achieves a rendering time of 0.008s and a PSNR of 29.38 [S1].

### HeadGaS Ablation (Change All Parameters)
An ablation variant of HeadGaS where all Gaussian parameters are changed/blended, implied by the name and context of other ablations like $\Delta(\mu, R)$. On the INSTA dataset with 97k Gaussians, it achieves a rendering time of 0.012s and a PSNR of 29.65 [S1].

### DualGS
A human-centric volumetric video representation using motion-aware joint Gaussians for global motion and appearance-aware skin Gaussians for visual detail. It employs a coarse-to-fine optimization strategy: Stage 1 optimizes joint Gaussians using a locally as-rigid-as-possible regularizer; Stage 2 jointly optimizes joint and skin Gaussians. It achieves a 120-fold compression ratio for immersive VR experiences [S2, unofficial].

## Operating characteristics
*   **Performance:** On the INSTA dataset, "HeadGaS change all" uses 97k Gaussians, has a rendering time of 0.012s, and achieves a PSNR of 29.65 [S1].
*   **Comparison:** It uses fewer Gaussians than the primary HeadGaS method (28k vs 97k? Note: The table in [S1] lists "Ours" as 28k Gaussians, 0.004s, 32.50 PSNR, and "Ours change all" as 97k Gaussians, 0.012s, 29.65 PSNR. This suggests "Ours change all" is less efficient in terms of Gaussian count and PSNR compared to the primary method, but more efficient than "Ours w/o blending" (135k) and "Ours w/o MLP" (295k) [S1]).
*   **Strengths:** Provides a baseline for evaluating the impact of the full blending mechanism on Gaussian parameters [S1].
*   **Limitations:** Lower PSNR and higher Gaussian count compared to the primary HeadGaS method [S1].

## Background
"HeadGaS change all" is an ablation study component of the HeadGaS method, which was introduced by Dhamo et al. in 2024 [S1]. The method is part of a series of ablations designed to understand the contributions of different components of the HeadGaS architecture, such as the blending mechanism and the MLP structure [S1].

## Sources
[S1] HeadGaS: Real-Time Animatable Head Avatars via 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-72627-9_26, fetched 2026-10-27
[S2] Dynamic Scene Reconstruction: Recent Advance in Real-time Rendering and Streaming, https://doi.org/10.48550/arXiv.2503.08166, fetched 2026-10-29 (unofficial)
