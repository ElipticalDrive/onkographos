# Error-state Camera Optimization

Also called: Camera Parameterization, SE(3) Error-state Optimization, Tangent Plane Camera Optimization, Tangent space camera update

## Overview
Error-state camera optimization refines camera poses and intrinsic parameters by optimizing residual corrections in the tangent space of the SE(3) manifold rather than updating the full transformation directly [S1, unofficial]. It builds on [Lie Algebra (Tangent Plane) Pose Optimization](14005-lie-algebra-tangent-plane-pose-optimization.md) by specifically handling both extrinsic pose errors and intrinsic focal length perturbations within a differentiable rendering framework [S1, unofficial]. This approach is distinct from direct manifold optimization by linearizing the update around a fixed initialization, ensuring smoother convergence for small residuals [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Initial camera poses $\mathbf{T}_c^{w,(0)}$ (fixed), initial focal lengths $\mathbf{f}^{(0)}$, principal point $(c_x, c_y)$ (fixed), pixel coordinates $(u, v)$, and rendered/observed images $\mathbf{C}, \mathbf{C}^*$.
*   **Outputs:** Optimized error states $\boldsymbol{\xi}$ (pose) and $\boldsymbol{\phi}_f$ (focal lengths).

**Data Structures:**
*   **Pose Error State:** $\boldsymbol{\xi} = [\boldsymbol{\xi}_r, \boldsymbol{\xi}_t] \in \mathbb{R}^6$, where $\boldsymbol{\xi}_r \in \mathbb{R}^3$ is the rotation perturbation and $\boldsymbol{\xi}_t \in \mathbb{R}^3$ is the translation perturbation.
*   **Focal Error State:** $\boldsymbol{\phi}_f \in \mathbb{R}^2$, representing the log-space correction for focal lengths.
*   **Intrinsic Matrix:** $\mathbf{K} = \begin{bmatrix} f_x & 0 & c_x \\ 0 & f_y & c_y \\ 0 & 0 & 1 \end{bmatrix}$.

**Steps:**
1.  **Initialize:** Set fixed odometry initialization $\mathbf{T}_c^{w,(0)}$ and initial focal lengths $\mathbf{f}^{(0)}$. Fix principal point $(c_x, c_y)$.
2.  **Compute Ray Directions:** For a pixel $(u, v)$, compute the camera-space ray direction:
    $$ \mathbf{r}^{\text{cam}} \propto \mathbf{K}^{-1} \begin{bmatrix} u \\ v \\ 1 \end{bmatrix} $$
    where $\mathbf{K}$ is updated using the current log-space focal estimate: $\mathbf{f} = \exp(\log \mathbf{f}^{(0)} + \boldsymbol{\phi}_f)$.
3.  **Differentiable Rendering:** Render the image $\mathbf{C}$ using the current pose $\mathbf{T}_c^w$ and intrinsics $\mathbf{K}$. The pose is computed as:
    $$ \mathbf{T}_c^w = \exp(\boldsymbol{\xi}^\wedge) \mathbf{T}_c^{w,(0)} $$
    where $\exp(\cdot)$ is the exponential map from se(3) to SE(3) and $(\cdot)^\wedge$ is the hat operator mapping $\mathbb{R}^6$ to se(3) [S1, unofficial].
4.  **Compute Loss:** Calculate the photometric loss $\mathcal{L}_{\text{cam}}$ between rendered image $\mathbf{C}$ and observed image $\mathbf{C}^*$:
    $$ \mathcal{L}_{\text{cam}} = \mathcal{L}_{\text{rgb}}(\mathbf{C}, \mathbf{C}^*) + \lambda \dots $$
    (Note: The specific regularization term is truncated in the source [S1, unofficial]).
5.  **Backpropagation:** Backpropagate the loss through the differentiable renderer to compute gradients with respect to $\boldsymbol{\xi}$ and $\boldsymbol{\phi}_f$. The trilinear density and vertex-interpolated DC appearance provide dense gradients with respect to camera parameters [S1, unofficial].
6.  **Update:** Update the error states $\boldsymbol{\xi}$ and $\boldsymbol{\phi}_f$ using an optimizer (e.g., Levenberg-Marquardt or Adam). The final pose and intrinsics are reconstructed from the error states and fixed initialization.

**Parameters:**
*   $\lambda$: Weight for regularization terms (value not specified in sources).
*   $\mathbf{T}_c^{w,(0)}$: Fixed odometry initialization.

**Edge Cases and Pitfalls:**
*   **Numerical Stability:** Focal lengths are optimized in log-space ($\log \mathbf{f}$) to ensure positivity and stability [S1, unofficial].
*   **Fixed Initialization:** The method assumes the initial poses $\mathbf{T}_c^{w,(0)}$ are metrically plausible. Large initial errors may lead to poor convergence as the error-state approximation is local [S1, unofficial].
*   **Principal Point:** The principal point $(c_x, c_y)$ is fixed at its initialized value and not optimized [S1, unofficial].

## Operating characteristics
*   **Complexity:** Depends on the differentiable renderer and the number of pixels/voxels. The error-state formulation reduces the optimization problem to a local linear update, improving convergence speed compared to global SE(3) optimization [S1, unofficial].
*   **Strengths:** Handles residual odometry errors effectively; provides dense gradients for camera parameters via differentiable rendering; numerically stable focal length optimization [S1, unofficial].
*   **Limitations:** Relies on good initialization; principal point cannot be optimized; assumes a shared pinhole model per intrinsic group [S1, unofficial].
*   **When to Use:** When refining poses and focal lengths in a differentiable volume rendering pipeline (e.g., SpaRV) where initial poses are known but have small errors [S1, unofficial].

## Background
Error-state camera optimization is used in systems like SpaRV (Sparse Radiance Voxels) for scalable geometrically consistent LiDAR-visual mapping and rendering [S1, unofficial]. It alternates camera optimization with map training to sharpen textures [S1, unofficial]. The method leverages the tangent space of SE(3) to handle pose corrections, a technique common in visual odometry and SLAM [S1, unofficial].

## Sources
[S1] SpaRV: Sparse Radiance Voxels for Scalable Geometrically Consistent LiDAR-Visual Mapping and Rendering, https://doi.org/10.48550/arxiv.2610.25665, fetched 2026-10-26 (unofficial)
