# Deformation Field Modeling

Also called: Deformation Fields, Canonical Space Mapping, Dynamic Coordinate Warping

## Overview
Deformation Field Modeling addresses the problem of representing a 4D spatiotemporal volume $D(x, y, z, t)$ using input images captured at different times $t$ by mapping dynamic coordinates to a static canonical space [S1, S3]. It differs from explicit attribute parameterization (e.g., [Dynamic 3D Gaussian Splatting](9168-dynamic-3d-gaussian-splatting-katsumata-et-al.md)) by using a neural network to predict displacement vectors, allowing for complex, non-rigid, and non-periodic motion that explicit functions cannot easily capture [S1, S3].

## Shared principles
All methods in this family address the problem of representing a 4D spatiotemporal volume $D(x, y, z, t)$ using a set of input images captured at different times $t$. The core mechanism involves decomposing the scene into static and dynamic components or modeling the temporal evolution of attributes directly.

1.  **Temporal Parameterization:** Attributes that vary with time (position $\mathbf{c}$, rotation $\mathbf{q}$, opacity $\alpha$, density $\sigma$) are expressed as functions of time $t$. Common functional forms include:
    *   **Polynomial/Linear:** $\mathbf{q}(t) = \mathbf{q}_0 + \mathbf{m}t$ [S2].
    *   **Fourier Series:** $\mathbf{c}(t) = \mathbf{c}_0 + \sum_{k=0}^{K} (\mathbf{a}_k \cos(k \omega t) + \mathbf{b}_k \sin(k \omega t))$ [S2].
    *   **Neural Deformation:** A neural network $\Psi_t(x, t)$ predicts displacement $\Delta x$ to warp points to a canonical space [S1, S3].
    *   **Implicit Fields:** Time-dependent NeRFs map $(x, d, t)$ to density and color, often using local neighborhoods of frames [S4].

2.  **Canonical Space Mapping:** Many methods (e.g., D-NeRF, Nerfies) introduce a canonical configuration (usually at $t=0$) where the scene is static. A deformation network maps dynamic coordinates $x$ at time $t$ to canonical coordinates $x'$:
    $$ x' = x + \Psi_t(x, t) $$
    The radiance field is then queried in the canonical space: $[\sigma, c] = \Psi_x(x', d)$ [S1, S3].

3.  **Rendering with Dynamics:** The rendering process integrates the time-dependent attributes. For Gaussians, the position and rotation are computed for the specific time $t$ of the camera ray before rasterization [S2]. For NeRFs, the density $\sigma$ and color $c$ are queried at the deformed or time-conditioned point [S1, S4].

4.  **Loss Functions:** Optimization typically minimizes photometric loss (L1 or LPIPS) between rendered and ground truth images [S1, S2]. Additional constraints often include:
    *   **Temporal Smoothness:** Penalizing abrupt changes in attributes between adjacent frames [S1].
    *   **Rigid/Flow Constraints:** Using optical flow or KNN rigid loss to guide deformation [S1].
    *   **Regularization:** Elastic or background regularization to prevent overfitting in deformation fields [S5].

## Subtopics

### [Dynamic Object Embedding in Static Scenes](8796-dynamic-object-embedding-in-static-scenes.md)
This method addresses the problem of integrating dynamic advertising objects into static NeRF scenes while maintaining perspective consistency and computational efficiency. It differs from traditional dynamic NeRF approaches (e.g., DyNeRF) by decoupling the scene into a static background and a localized dynamic region, using a lightweight deformation field to warp coordinates rather than training independent models for each time step. This reduces parameter count by 95% compared to full-scene dynamic models, making it suitable for VR devices. See [Dynamic Object Embedding in Static Scenes](8796-dynamic-object-embedding-in-static-scenes.md).

### [Dynamic 3D Gaussian Splatting (Katsumata et al.)](9168-dynamic-3d-gaussian-splatting-katsumata-et-al.md)
Dynamic 3D Gaussian Splatting (Katsumata et al.) models dynamic scenes by parameterizing the temporal evolution of Gaussian positions and orientations using explicit mathematical functions, while keeping other Gaussian attributes (such as color, opacity, and scale) static. It distinguishes itself from implicit deformation networks by using a Fourier series for position and linear functions for rotation, ensuring temporal continuity and robustness against overfitting. This approach is particularly suited for scenes with periodic or smooth motion, such as human articulation. See [Dynamic 3D Gaussian Splatting (Katsumata et al.)](9168-dynamic-3d-gaussian-splatting-katsumata-et-al.md).

### NeRF with Deformation Fields (Nerfies)
A NeRF model built using a deformation field to improve performance in the presence of non-rigid transformations (dynamic scenes) [S1]. It introduces an additional MLP that maps input observation frame coordinates to deformed canonical coordinates [S1]. It adds elastic regularization, background regularization, and coarse-to-fine deformation regularization by adaptive masking the positional encoding [S1].

## Background
The concept of using deformation fields for dynamic scenes was introduced by Park et al. in November 2020 with Nerfies [S1]. This work strongly improved performance in the presence of non-rigid transformations, such as dynamic scenes, which baseline NeRF failed to reconstruct accurately [S1]. Concurrent to Nerfies was NerFace (December 2020), which used per-frame learned latent codes and facial expression coefficients [S1]. Subsequently, Park et al. introduced HyperNeRF (June 2021), which built on Nerfies [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
