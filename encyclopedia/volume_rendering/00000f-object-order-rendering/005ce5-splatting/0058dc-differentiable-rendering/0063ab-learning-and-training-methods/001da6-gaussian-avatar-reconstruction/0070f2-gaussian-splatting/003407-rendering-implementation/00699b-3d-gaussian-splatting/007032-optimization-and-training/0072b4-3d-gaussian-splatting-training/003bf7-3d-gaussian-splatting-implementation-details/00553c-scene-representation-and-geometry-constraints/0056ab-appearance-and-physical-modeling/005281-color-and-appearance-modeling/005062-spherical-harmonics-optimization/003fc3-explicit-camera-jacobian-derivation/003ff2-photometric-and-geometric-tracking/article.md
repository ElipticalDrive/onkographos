# Photometric and Geometric Tracking

Also called: SLAM Tracking, Pose Optimization

## Overview
Photometric and Geometric Tracking optimizes only the current camera pose without updating the map representation, minimizing photometric and/or geometric residuals to estimate motion [S1, unofficial]. It sets itself apart from full SLAM or mapping methods by decoupling pose estimation from scene reconstruction, allowing for efficient real-time tracking using pre-existing Gaussian splats [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 3D Gaussians $\mathcal{G}$ (fixed), the current estimated camera pose $\mathbf{T}_{CW}$, observed image $\bar{I}$, and optionally observed depth $\bar{D}$.
*   **Outputs:** Optimized camera pose $\mathbf{T}_{CW}$.

**Data Structures:**
The Gaussian set $\mathcal{G}$ consists of $N$ Gaussians. Each Gaussian $i$ is defined by mean $\mu_W$, covariance $\Sigma$, opacity $\alpha_i$, and spherical harmonic coefficients for color. The pose $\mathbf{T}_{CW} \in SE(3)$ transforms world coordinates to camera coordinates.

**Steps:**
1.  **Render Depth:** For each pixel, compute the expected depth $\mathcal{D}_p$ by alpha-blending the distances $z_i$ of visible Gaussians along the camera ray [S1, unofficial].
    $$ \mathcal{D}_p = \sum_{i \in \mathcal{N}} z_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    where $z_i$ is the distance to the mean $\mu_W$ of Gaussian $i$ along the camera ray, and $\mathcal{N}$ is the set of sorted Gaussians contributing to the pixel [S1, unofficial].
2.  **Compute Photometric Residual:** Render the image $I(\mathcal{G}, \mathbf{T}_{CW})$ from the current pose and compare it to the observed image $\bar{I}$ using L1 norm [S1, unofficial].
    $$ E_{pho} = \left\| I(\mathcal{G}, \mathbf{T}_{CW}) - \bar{I} \right\|_1 $$
3.  **Compute Geometric Residual (Optional):** If depth observations $\bar{D}$ are available, render depth $D(\mathcal{G}, \mathbf{T}_{CW})$ and compare to $\bar{D}$ [S1, unofficial].
    $$ E_{geo} = \left\| D(\mathcal{G}, \mathbf{T}_{CW}) - \bar{D} \right\|_1 $$
4.  **Combine Losses:** Minimize the weighted sum of residuals. Let $\lambda_{pho}$ be a hyperparameter balancing the terms [S1, unofficial].
    $$ E = \lambda_{pho} E_{pho} + (1 - \lambda_{pho}) E_{geo} $$
5.  **Optimize Pose:** Use analytical Jacobians of the camera pose with respect to the Gaussian splats to minimize $E$ via gradient descent [S1, unofficial].
6.  **Apply Regularization:** Penalize non-edge or low-opacity pixels to improve robustness [S1, unofficial]. Optimize affine brightness parameters to handle varying exposure [S1, unofficial].

**Parameters:**
*   $\lambda_{pho}$: Weight for photometric loss. Typical value not specified in sources.
*   Affine brightness parameters: Optimized per frame to handle exposure changes [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Low Opacity/Non-Edge Pixels:** These can introduce noise; the method explicitly penalizes them [S1, unofficial].
*   **Exposure Changes:** Handled by optimizing affine brightness parameters [S1, unofficial].
*   **Depth Availability:** Geometric loss is optional and requires synchronized depth sensors [S1, unofficial].

## Subtopics

### Incremental Pose and Feature Optimization
This variant, used in hierarchical implicit grid systems, alternates between tracking (optimizing pose) and mapping (optimizing features) [S2, unofficial]. During tracking, only the current robot pose $T_{ks}$ is optimized while submap features $F$ are kept fixed [S2, unofficial]. It employs a trust-region regularization term $\rho(T_b, T) = w_\rho \max(\| \text{Log}(T_b^{-1} T) \|_2 - \tau, 0)$ to penalize pose updates larger than the trust-region radius $\tau$, where $w_\rho$ is a weight and $\text{Log}$ is the logarithmic map on $SE(3)$ [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of Gaussians and pixels. Analytical Jacobians avoid the overhead of automatic differentiation [S1, unofficial].
*   **Memory Complexity:** Low, as the map representation $\mathcal{G}$ is not updated during tracking [S1, unofficial].
*   **Strengths:** Efficient real-time tracking; robust to exposure changes via brightness optimization; combines photometric and geometric cues [S1, unofficial].
*   **Limitations:** Requires a pre-existing map or keyframes; drift may accumulate if not constrained by keyframing or loop closure [S1, unofficial].
*   **When to Use:** Real-time SLAM tracking where map updates are decoupled from pose estimation [S1, unofficial].
*   **When Not to Use:** When the map is unknown or requires frequent updates [S1, unofficial].

## Background
Originated in the context of Gaussian Splatting SLAM, where tracking is separated from mapping to enable efficient online operation [S1, unofficial]. The method builds on differentiable rendering of 3D Gausslets and analytical Jacobian derivation for SE(3) poses [S1, unofficial].

## Sources
[S1] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
[S2] HIGS: Hierarchical Implicit Grids for Joint Geometric and Semantic Scene Understanding, https://doi.org/10.48550/arxiv.2609.38620, fetched 2026-10-26 (unofficial)
