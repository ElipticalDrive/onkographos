# Covariance Factorisation

Also called: Covariance Decomposition, RQ Decomposition for Covariance, Rotation and Scale Parameterization

## Overview
Covariance Factorisation is a parameterization technique for 3D Gaussian splats that decomposes the covariance matrix $\Sigma$ into a rotation matrix $\mathbf{R}$ and a scaling matrix $\mathbf{S}$ [S1, unofficial][S2]. This method ensures the covariance matrix remains positive semi-definite (PSD) during unconstrained gradient descent optimization, which is critical for maintaining the physical validity of the Gaussian primitives [S1, unofficial][S2]. It sets itself apart from direct covariance optimization by explicitly decoupling rotation and scale, allowing for intuitive parameterization and stable gradient derivation [S2].

## How it works
**Inputs and Outputs:**
- **Input:** Unconstrained learnable parameters for rotation and scale.
- **Output:** A valid, positive semi-definite 3D covariance matrix $\Sigma$.

**Data Structures:**
The method relies on two primary learnable components per Gaussian splat:
1.  **Rotation Matrix $\mathbf{R}$:** A $3 \times 3$ orthogonal matrix. In practice, this is often parameterized by a unit quaternion or Euler angles to ensure orthogonality, though the factorisation itself assumes $\mathbf{R}$ is a valid rotation matrix [S2].
2.  **Scaling Matrix $\mathbf{S}$:** A diagonal matrix $\mathbf{S} = \text{diag}(s_1, s_2, s_3)$, where $s_i$ are the scale factors along the principal axes. The scale vector $\mathbf{s} \in \mathbb{R}^3$ is typically constrained to be positive ($s_i > 0$) to maintain physical meaning [S2, S4].

**Steps:**
1.  **Parameter Initialization:** Initialize the rotation parameters (e.g., quaternion) and scale parameters (e.g., log-scale or raw positive values) for each Gaussian splat.
2.  **Optimization:** Perform unconstrained gradient descent on these parameters. Because the parameters are not the covariance entries directly, there is no risk of generating a non-PSD matrix during updates [S1, unofficial].
3.  **Reconstruction:** At each rendering or update step, reconstruct the covariance matrix using the factorisation formula [S2]:
    $$
    \boldsymbol{\Sigma} = \mathbf{R} \mathbf{S} \mathbf{S}^T \mathbf{R}^T
    $$
    Here, $\mathbf{S} \mathbf{S}^T$ is a diagonal matrix with entries $s_i^2$. Thus, $\boldsymbol{\Sigma} = \mathbf{R} \text{diag}(s_1^2, s_2^2, s_3^2) \mathbf{R}^T$ [S2, S4].
4.  **Usage:** The resulting $\boldsymbol{\Sigma}$ is used for EWA projection and rendering [S1, unofficial].

**Formulas:**
-   **Covariance Reconstruction:**
    $$
    \boldsymbol{\Sigma} = \mathbf{R} \mathbf{S} \mathbf{S}^T \mathbf{R}^T
    $$
    Where:
    -   $\boldsymbol{\Sigma}$ is the $3 \times 3$ covariance matrix.
    -   $\mathbf{R}$ is the $3 \times 3$ rotation matrix.
    -   $\mathbf{S}$ is the $3 \times 3$ diagonal scaling matrix.
    -   $\mathbf{S}^T$ is the transpose of $\mathbf{S}$ (equal to $\mathbf{S}$ for diagonal matrices).

**Parameters:**
-   **Scale Vector $\mathbf{s}$:** Typically optimized in log-space to ensure positivity, or constrained via activation functions [S1, unofficial].
-   **Rotation Representation:** Unit quaternions are common to ensure $\mathbf{R}$ remains orthogonal [S2].

**Edge Cases and Pitfalls:**
-   **Non-PSD Matrices:** Direct optimization of $\Sigma$ entries can lead to non-PSD matrices, causing numerical instability and invalid Gaussian shapes. Factorisation avoids this by construction [S1, unofficial][S2].
-   **Degenerate Scales:** If any scale $s_i$ becomes zero or negative (if not constrained), the Gaussian becomes degenerate. Constraints or activations (e.g., softplus, exp) are required [S1, unofficial].
-   **Rotation Ambiguity:** The factorisation is not unique if signs of scales and rotation axes are not handled consistently. Standard practice assumes positive scales and a proper rotation matrix [S2].

## Subtopics

### [Cholesky Parameterisation for Gaussian Splats](3120-cholesky-parameterisation-for-gaussian-splats.md)
Cholesky Parameterisation represents the covariance matrix using a lower triangular Cholesky factor, guaranteeing positive-definiteness by construction. It uses $D(D+1)/2$ elements per splat (6 for 3D) and applies a softplus activation with a minimum standard deviation floor $\sigma_{min} = 1/\sqrt{12}$ voxels to the diagonal, while bounding amplitudes with a configurable maximum $a_{max}$. Unlike Covariance Factorisation, which decouples rotation and scale, Cholesky uses a triangular factorization, which may be less intuitive for interpreting ellipsoid geometry but offers similar stability guarantees.

### [Quaternion-Scale Covariance Parameterization](11801-quaternion-scale-covariance-parameterization.md)
This method parameterizes the 3D Gaussian covariance matrix $\Sigma$ using a unit quaternion $q$ and a 3D scale vector $s$ to ensure the matrix remains positive semi-definite during optimization. It solves the problem that directly optimizing the entries of $\Sigma$ can result in non-positive semi-definite matrices, violating physical interpretation and causing numerical instability. Unlike siblings such as [Cholesky Parameterisation for Gaussian Splats](3120-cholesky-parameterisation-for-gaussian-splats.md), which uses a triangular factorization, this approach explicitly decouples rotation and scale, allowing for efficient gradient derivation and intuitive ellipsoid parameterization.

### Gamma-Decomposed Covariance Update
A numerical stability technique for computing the new covariance matrix after recycling, replacing the unsupported $\beta()$ function with $\Gamma()$ and $\ln(\Gamma())$ to prevent overflow [S3]. This variant addresses computational issues in specific update rules for covariance matrices during optimization.

### Covariance Matrix Normalization
A technique to transform a covariance matrix $\Sigma$ into a normalized form $\hat{\Sigma}$ by dividing it by its trace, separating the scaling vector $s$ from the rotation matrix $R$ [S4]. The formula is:
$$
\hat{\Sigma} = \frac{\Sigma}{\text{Tr}(\Sigma)} = \mathbf{R} \frac{\mathbf{S}^2}{\|s\|_2^2} \mathbf{R}^T = \mathbf{R} \hat{\mathbf{S}}^2 \mathbf{R}^T
$$
where $\|s\|_2^2 = \text{Tr}(\Sigma)$ [S4]. This normalization ensures the scaling vector has a unit norm, which can be useful for compression or clustering applications.

## Operating characteristics
-   **Time Complexity:** The reconstruction of $\Sigma$ from $\mathbf{R}$ and $\mathbf{S}$ is $O(1)$ per splat (constant number of multiplications for $3 \times 3$ matrices).
-   **Memory Complexity:** Stores rotation (e.g., 4 floats for quaternion) and scale (3 floats) parameters, totaling 7 floats per splat, compared to 6 floats for the symmetric covariance matrix. However, the overhead is minimal and justified by stability.
-   **Strengths:** Guarantees positive semi-definiteness, enabling stable unconstrained optimization. Intuitive interpretation of rotation and scale.
-   **Limitations:** Requires reconstruction step at each iteration. Parameterization of rotation (e.g., quaternion) may introduce singularities or require normalization steps.
-   **When to Use:** Standard choice for 3D Gaussian Splatting and similar radiance-field optimization methods where stable gradient descent is required.

## Background
Covariance Factorisation is a core component of 3D Gaussian Splatting (3DGS), introduced by Kerbl et al. in 2023 [S1, unofficial]. It was developed to address the instability of directly optimizing covariance matrices in differentiable rendering. The method leverages the fact that any positive semi-definite matrix can be decomposed into a rotation and a scaling component, a concept rooted in linear algebra and ellipsoid parameterization. It has since been adopted and adapted in various extensions of 3DGS, such as EVolSplat and compressed variants [S2, S4].

## Sources
[S1] An Architectural Review of the 3D Gaussian Splatting Rendering Pipeline, https://doi.org/10.5281/zenodo.21739653, fetched 2026-10-26 (unofficial)
[S2] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S3] 3D Student Splatting and Scooping, https://doi.org/10.1109/cvpr52734.2025.01960, fetched 2026-10-29
[S4] Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.00985, fetched 2026-10-26
