# Explicit Camera Jacobian Derivation

Also called: Analytical Jacobians for 3DGS, Camera Pose Jacobian Derivation (Lie Group), Lie Algebra Jacobian for 3DGS

## Overview
This method derives analytical Jacobians of the SE(3) camera pose with respect to 3D Gaussian Splatting (3DGS) primitives to enable efficient gradient-based camera tracking. It solves the problem of high computational overhead associated with automatic differentiation in differentiable rasterization, which is critical for real-time Simultaneous Localization and Mapping (SLAM) systems [S1, unofficial]. Unlike prior ray-intersection methods that assume a small number of Gaussians, this approach is tailored for the EWA splatting rasterizer used in 3DGS, providing minimal Jacobians via Lie algebra that match the degrees of freedom of the camera pose [S1, unofficial].

## How it works
The method computes the derivatives of the 2D Gaussian projection parameters (mean $\mu_I$ and covariance $\Sigma_I$) with respect to the camera-to-world pose $T_{CW} \in SE(3)$. It uses the chain rule to decompose these derivatives into terms involving the Gaussian mean in camera coordinates $\mu_C$, the projection Jacobian $J$, and the rotational component $W$ of the camera pose [S1, unofficial].

**Inputs:**
- Camera pose $T_{CW} \in SE(3)$, parameterized by a translation vector and a rotation matrix $R_{CW}$.
- 3D Gaussian mean in world coordinates $\mu_W$.
- 3D Gaussian covariance in world coordinates $\Sigma_W$.
- Camera intrinsic matrix $K$.
- Rotation matrix component $W$ derived from $R_{CW}$ (specifically, the upper-left $3 \times 3$ block).

**Outputs:**
- Jacobian $\frac{\partial \mu_I}{\partial T_{CW}} \in \mathbb{R}^{2 \times 6}$.
- Jacobian $\frac{\partial \Sigma_I}{\partial T_{CW}} \in \mathbb{R}^{4 \times 6}$ (for the 4 unique elements of the symmetric $2 \times 2$ covariance matrix).

**Data Structures:**
- $T_{CW}$: Represented as a 6-dimensional Lie algebra vector $\xi = [\rho, \theta]^T \in \mathbb{R}^6$, where $\rho$ is translation and $\theta$ is rotation vector.
- Skew-symmetric matrix operator $(\cdot)^\times$: Maps a 3D vector to a $3 \times 3$ skew-symmetric matrix.

**Steps:**

1.  **Define Manifold Derivative:**
    The derivative of a function $f(T)$ on the Lie group $SE(3)$ is defined using the left-trivialized tangent space [S1, unofficial]:
    $$
    \frac{\mathcal{D}(f(\mathbf{T}))}{\mathcal{D}\mathbf{T}} \triangleq \lim_{\tau \rightarrow 0} \frac{\text{Log}(f(\text{Exp}(\tau) \circ \mathbf{T}) \circ f(\mathbf{T})^{-1})}{\tau}
    $$
    where $\tau \in \mathbb{R}^6$ is an infinitesimal perturbation in the Lie algebra $se(3)$, $\text{Exp}$ maps algebra to group, $\text{Log}$ maps group to algebra, and $\circ$ is group composition.

2.  **Compute Mean Jacobian:**
    The mean in camera coordinates is $\mu_C = R_{CW} \mu_W + t_{CW}$. The derivative with respect to the camera pose perturbation $\tau = [\rho, \theta]^T$ is derived as [S1, unofficial]:
    $$
    \frac{\mathcal{D}\mu_C}{\mathcal{D}T_{CW}} = [I \quad -\mu_C^\times]
    $$
    Derivation details:
    $$
    \frac{\mathcal{D}\mu_C}{\mathcal{D}T_{CW}} = \lim_{\tau \rightarrow 0} \frac{\text{Exp}(\tau) \cdot \mu_C - \mu_C}{\tau} = \lim_{\tau \rightarrow 0} \frac{(\mathbf{I} + \tau^\times) \cdot \boldsymbol{\mu}_C - \boldsymbol{\mu}_C}{\tau} = \begin{bmatrix} I & -\mu_C^\times \end{bmatrix}
    $$
    where $\tau^\times = \rho^\times + \theta^\times$ is the skew-symmetric matrix of the combined perturbation.

3.  **Compute Rotation Component Jacobian:**
    The derivative of the rotation matrix component $W$ (which is $R_{CW}$) with respect to the rotation part of the pose is required for the covariance derivative. Let $W_{:,i}$ be the $i$-th column of $W$. The derivative with respect to the full pose $T_{CW}$ is [S1, unofficial]:
    $$
    \frac{\mathcal{D}W}{\mathcal{D}T_{CW}} = \begin{bmatrix} 0 & -W_{:,1}^\times \\ 0 & -W_{:,2}^\times \\ 0 & -W_{:,3}^\times \end{bmatrix}
    $$
    This is derived from $\frac{\partial W}{\partial \theta_i} = e_i^\times W$, where $e_i$ is the $i$-th basis vector.

4.  **Assemble Full Jacobians:**
    Using the chain rule, the final Jacobians are computed as [S1, unofficial]:
    $$
    \frac{\partial \mu_I}{\partial T_{CW}} = \frac{\partial \mu_I}{\partial \mu_C} \frac{\mathcal{D}\mu_C}{\mathcal{D}T_{CW}}
    $$
    $$
    \frac{\partial \Sigma_I}{\partial T_{CW}} = \frac{\partial \Sigma_I}{\partial J} \frac{\partial J}{\partial \mu_C} \frac{\mathcal{D}\mu_C}{\mathcal{D}T_{CW}} + \frac{\partial \Sigma_I}{\partial W} \frac{\mathcal{D}W}{\mathcal{D}T_{CW}}
    $$
    Here, $\frac{\partial \mu_I}{\partial \mu_C}$, $\frac{\partial J}{\partial \mu_C}$, and $\frac{\partial \Sigma_I}{\partial J}$ are standard derivatives from the 3DGS rasterization pipeline (projection and covariance transformation). $\frac{\partial \Sigma_I}{\partial W}$ accounts for the rotation of the 3D covariance $\Sigma_W$ into the camera frame.

**Parameters:**
- $\mu_C \in \mathbb{R}^3$: Gaussian center in camera coordinates.
- $T_{CW} \in SE(3)$: Camera-to-world pose.
- $W \in \mathbb{R}^{3 \times 3}$: Rotation matrix component of $T_{CW}$.
- $\theta \in \mathbb{R}^3$: Rotation vector.
- $\tau \in \mathbb{R}^6$: Infinitesimal Lie algebra parameter.
- $\times$: Hat operator mapping vector to skew-symmetric matrix.

**Edge Cases and Pitfalls:**
- **Precision:** The use of Lie algebra ensures the Jacobian dimensionality matches the 6 degrees of freedom, avoiding redundant computations inherent in matrix representations [S1, unofficial].
- **Boundary Conditions:** The derivation assumes the camera pose is well-defined and the exponential map is valid. For large perturbations, the linear approximation in the limit $\tau \rightarrow 0$ holds only for the derivative calculation, not for the pose update itself.

## Subtopics

### Photometric and Geometric Tracking
This variant applies the explicit Jacobians to optimize only the current camera pose without updating the map representation, minimizing photometric and/or geometric residuals [S1, unofficial]. It is chosen for real-time tracking where the scene geometry is assumed static or pre-mapped.
- **Photometric Residual:** $E_{pho} = \| I(\mathcal{G}, \mathbf{T}_{CW}) - \bar{I} \|_1$, where $I(\mathcal{G}, \mathbf{T}_{CW})$ renders the Gaussians $\mathcal{G}$ from pose $\mathbf{T}_{CW}$ and $\bar{I}$ is the observed image [S1, unofficial].
- **Geometric Residual:** $E_{geo} = \| D(\mathcal{G}, \mathbf{T}_{CW}) - \bar{D} \|_1$, where $D(\mathcal{G}, \mathbf{T}_{CW})$ is depth rasterization and $\bar{D}$ is observed depth [S1, unofficial].
- **Depth Rasterization:** $\mathcal{D}_p = \sum_{i \in \mathcal{N}} z_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j)$, where $z_i$ is the distance to the mean $\mu_W$ of Gaussian $i$ along the camera ray [S1, unofficial].
- **Optimization:** Minimizes $\lambda_{pho} E_{pho} + (1-\lambda_{pho}) E_{geo}$. Affine brightness parameters are optimized for exposure variations, and non-edge or low-opacity pixels are penalized [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The analytical derivation eliminates the overhead of automatic differentiation for the camera pose, making gradient computation comparable to the forward rasterization pass. This enables real-time performance (e.g., 3.2 FPS monocular, 2.5 FPS RGB-D on Replica dataset office2) [S1, unofficial].
- **Memory Complexity:** Minimal, as it only requires storing the Jacobian matrices ($2 \times 6$ and $4 \times 6$ per Gaussian) during the backward pass, similar to standard 3DGS.
- **Strengths:** Enables accurate tracking with at least 50 iterations of gradient descent per frame, which is feasible due to the computational efficiency of the explicit Jacobians [S1, unofficial]. It unifies tracking and mapping within the 3DGS representation.
- **Limitations:** Requires the rasterization pipeline to be differentiable with respect to the camera pose, which is not always the case in standard 3DGS implementations without modification. It is designed for SLAM applications and may not be necessary for static novel view synthesis.

## Background
The method originates from the "Gaussian Splatting SLAM" paper, which introduced 3D Gaussian Splatting as a unified representation for SLAM, including tracking, mapping, and keyframe management [S1, unofficial]. Prior methods for camera pose optimization with 3D Gaussians assumed a smaller number of primitives and relied on ray-intersection, which is not applicable to the EWA splatting rasterizer used in 3DGS [S1, unofficial]. This work provides the first analytical Jacobian of SE(3) camera pose with respect to 3D Gaussians in the context of EWA splatting, enabling efficient gradient-based optimization for dynamic tracking and 4D scene representation [S1, unofficial].

## Sources
[S1] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
