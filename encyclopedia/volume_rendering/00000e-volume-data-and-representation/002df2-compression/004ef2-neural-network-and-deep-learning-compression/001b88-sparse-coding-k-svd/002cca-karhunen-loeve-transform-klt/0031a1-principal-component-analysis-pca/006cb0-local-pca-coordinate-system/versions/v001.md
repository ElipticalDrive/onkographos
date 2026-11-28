# Local-PCA Coordinate System

Also called: PCA-based Local Coordinate System

## Overview
The Local-PCA Coordinate System constructs a local basis for each initialized center in a 3D Gaussian reconstruction pipeline by performing Principal Component Analysis on its nearest neighbors [S1, unofficial]. It serves as a geometric anchor that adapts the projection constraints to the local structure (line-like vs. plane-like), preventing artifacts in texture-degraded scenes [S1, unofficial]. Unlike standard PCA which assumes global linearity, this method computes a fixed, unweighted covariance for each center to ensure the coordinate system remains stable and is not altered by subsequent volume rendering statistics or consensus weights [S1, unofficial].

## How it works
**Inputs:**
*   A set of initialized centers $\{x_0\}$.
*   For each center $x_0$, a fixed set of nearest neighbor indices $N$ constructed once from the shared initialized centers [S1, unofficial].

**Outputs:**
*   A local orthonormal basis $\{v_1, v_2, v_3\}$ for each center.
*   A projection matrix $P$ for structural constraints.
*   Structural confidence metrics $C$ and $\hat{C}$ [S1, unofficial].

**Data Structures:**
*   **Covariance Matrix $\Sigma$**: A $3 \times 3$ symmetric matrix.
*   **Eigenvalues $\lambda$**: A vector of size 3, sorted such that $\lambda_1 \le \lambda_2 \le \lambda_3$.
*   **Eigenvectors $v$**: A $3 \times 3$ matrix where columns are the unit eigenvectors $v_1, v_2, v_3$ corresponding to $\lambda_1, \lambda_2, \lambda_3$ [S1, unofficial].

**Steps:**

1.  **Compute Local Covariance:**
    Calculate the mean $\bar{x}$ of the neighbors $N$ and the covariance matrix $\Sigma$ without using any weights (e.g., VTC consensus weights) to prevent rendering statistics from altering the geometry [S1, unofficial]:
    $$ \Sigma = \frac{1}{|N|} \sum_{i \in N} (x_i - \bar{x})(x_i - \bar{x})^T $$
    [G] Ensure $|N| > 0$ and handle degenerate cases where neighbors are collinear or coincident by adding a small regularization term $\epsilon I$ if necessary, though the source defines $\epsilon$ only in the denominator of scores.

2.  **Eigendecomposition:**
    Perform eigendecomposition on $\Sigma$ to obtain eigenvalues and eigenvectors. Sort the eigenvalues in ascending order:
    $$ \lambda_1 \le \lambda_2 \le \lambda_3 $$
    The corresponding unit eigenvectors $v_1, v_2, v_3$ form the local-PCA coordinate system [S1, unofficial].
    *   $v_1$ (minimum eigenvalue) is treated as a local-normal candidate.
    *   $v_3$ (maximum eigenvalue) is treated as the principal-axis candidate for slender structures [S1, unofficial].

3.  **Compute Structural Scores:**
    Calculate linearity and planarity scores using a small constant $\epsilon$ (e.g., $10^{-6}$) to prevent division by zero [S1, unofficial]:
    $$ \rho_l = \frac{\lambda_3 - \lambda_2}{\lambda_3 + \epsilon} $$
    $$ \rho_p = \frac{\lambda_2 - \lambda_1}{\lambda_3 + \epsilon} $$
    [S1, unofficial]
    *   $\rho_l \to 1$ indicates a line-like structure.
    *   $\rho_p \to 0$ indicates a plane-like structure [S1, unofficial].

4.  **Construct Projection Matrix:**
    Build the allowed-direction projection matrix $P$ to interpolate between principal-axis preservation (for lines) and tangent-plane preservation (for planes) [S1, unofficial]:
    $$ P = v_3 v_3^T + (1 - \rho_l)(v_2 v_2^T + v_1 v_1^T) $$
    [S1, unofficial]

5.  **Compute Structural Confidence (Optional Variant):**
    Calculate raw confidence $C$ and modulated confidence $\hat{C}$ to determine if the PCA basis reliably represents the local structure [S1, unofficial]:
    $$ C = \left[ \max\left( \frac{\lambda_3 - \lambda_1}{\lambda_3 + \epsilon} \right) \right]^{\gamma} $$
    $$ \hat{C} = (1 - \beta) C + \beta w_{VTC} $$
    Where $\gamma$ is a confidence exponent and $\beta$ is a support-mixing coefficient incorporating the VTC consensus weight $w_{VTC}$ [S1, unofficial].

**Parameters:**
*   $\epsilon$: Small constant for numerical stability. Value not explicitly given in sources, typically $10^{-6}$ or similar [S1, unofficial].
*   $\gamma$: Confidence exponent. Value not explicitly given in sources [S1, unofficial].
*   $\beta$: Support-mixing coefficient. Value not explicitly given in sources [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Isotropic Neighborhoods:** If $\lambda_1 \approx \lambda_2 \approx \lambda_3$, the structure is isotropic. The scores $\rho_l$ and $\rho_p$ will be small, and confidence $C$ will be low. The method reverts to standard constraints to prevent artifacts [S1, unofficial].
*   **Weighting:** The covariance must be computed *without* weights. Using VTC consensus weights for covariance calculation is explicitly prohibited as it alters the local coordinate system [S1, unofficial].

## Subtopics

### Structural Confidence
A metric measuring whether the current PCA basis can reliably represent the true local structure by combining directional saliency, eigenvalue anisotropy, and observation support [S1, unofficial]. It uses a raw confidence $C$ based on eigenvalue spread and a modulated confidence $\hat{C}$ that incorporates VTC consensus weight $w_{VTC}$ via a mixing coefficient $\beta$ [S1, unofficial]. High confidence increases constraint strength; low confidence reverts to standard 3DGS forms [S1, unofficial].

## Operating characteristics
*   **Complexity:** The method requires computing a $3 \times 3$ covariance matrix and performing eigendecomposition for each center. This is $O(|N|)$ for covariance and constant time for decomposition per center.
*   **Memory:** Requires storing the local basis ($3 \times 3$ matrix) and scores per center.
*   **Strengths:** Provides geometric anchoring that adapts to local structure (lines vs. planes), improving reconstruction in texture-degraded scenes [S1, unofficial].
*   **Limitations:** Relies on a fixed neighborhood structure; if the neighborhood is poorly chosen (e.g., too small or noisy), the basis may be unreliable [S1, unofficial].
*   **When to use:** In 3D Gaussian reconstruction pipelines where texture degradation causes ambiguity in surface orientation, and where standard isotropic constraints lead to artifacts [S1, unofficial].

## Background
The Local-PCA Coordinate System was introduced in the context of Visibility-Aware Texture Consensus (VTC) for 3D Gaussian Reconstruction in texture-degraded scenes [S1, unofficial]. It addresses the limitation of standard 3D Gaussian Splatting which assumes isotropic uncertainty, by introducing anisotropic constraints based on local geometric structure [S1, unofficial].

## Sources
[S1] Visibility-Aware Texture Consensus and Local Structural Constraints for 3D Gaussian Reconstruction in Texture- Degraded Scenes, https://doi.org/10.21203/rs.3.rs-10727268/v1, fetched 2026-10-26 (unofficial)
