# Animatable Gaussians


## Overview
Animatable Gaussians is a method for reconstructing animatable 3D human avatars using explicit pose-conditioned offsets within an SMPL-X rig [S1, unofficial]. It solves the problem of generating dynamic, high-fidelity human representations from static or video data by decoupling rigid skeletal motion from non-rigid surface deformation. It is distinguished from siblings like [Gaussian Avatar](1479-gaussian-avatar.md) and [3DGS-Avatar](1480-3dgs-avatar.md) by its use of explicit, analytically defined pose-conditioned offset fields rather than implicit MLPs or UV-based parameterizations [S1, unofficial].

## How it works
**Inputs:** A set of multi-view images or video frames of a human subject; SMPL-X model parameters (shape $\beta$, pose $\theta$, expression $\psi$) fitted to the input data.
**Outputs:** A set of 3D Gaussians $\{g_i\}$ defined in the canonical (rest) pose, each with position $\mathbf{p}_i$, covariance $\Sigma_i$, opacity $\alpha_i$, spherical harmonic coefficients $\mathbf{c}_i$, and deformation network parameters.

**Data Structures:**
*   **Gaussian Attributes:** Stored in flat arrays. Each Gaussian $i$ has:
    *   $\mathbf{p}_i \in \mathbb{R}^3$: Canonical position.
    *   $\mathbf{R}_i \in \mathbb{R}^{3 \times 3}$: Rotation matrix (from quaternion).
    *   $\mathbf{s}_i \in \mathbb{R}^3$: Scale vector.
    *   $\alpha_i \in \mathbb{R}$: Opacity.
    *   $\mathbf{c}_i \in \mathbb{R}^k$: SH coefficients for color.
    *   $\mathbf{w}_i \in \mathbb{R}^d$: Weights for the deformation network.
*   **Deformation Network:** A lightweight MLP or linear layer mapping pose parameters to residual offsets.

**Steps:**
1.  **Initialization:** Initialize Gaussians on the surface of the SMPL-X mesh in the canonical rest pose ($\theta=0, \psi=0$) [S1, 1479-gaussian-avatar.md, 1480-3dgs-avatar.md, 6157-animatable-gaussians.md, unofficial].
2.  **Rigid Transformation:** For each frame with pose $\theta$, compute the rigid transformation $T_{rigid}(\mathbf{p}_i, \theta)$ using Linear Blend Skinning (LBS) derived from the SMPL-X skeleton [S1, unofficial].
3.  **Non-Rigid Deformation:** Predict the residual offset $\Delta \mathbf{p}_i$ using the pose-conditioned deformation network: $\Delta \mathbf{p}_i = f_{\text{deform}}(\mathbf{p}_i, \theta, \mathbf{w}_i)$. The network takes the canonical position, current pose, and Gaussian-specific weights as input.
4.  **Deformed Position:** Compute the final deformed position: $\mathbf{p}'_i = T_{rigid}(\mathbf{p}_i, \theta) + \Delta \mathbf{p}_i$.
5.  **Rendering:** Render the deformed Gaussians using differentiable splatting. The covariance $\Sigma'_i$ is updated to reflect the deformation (Jacobian of the transformation) [1479-gaussian-avatar.md, 1480-3dgs-avatar.md, 6157-animatable-gaussians.md].
6.  **Optimization:** Minimize photometric loss $\mathcal{L}_{rec}$ between rendered images and ground truth. Gradients flow through the deformation network to update $\mathbf{w}_i$ and network parameters.

**Formulas:**
*   Rigid Transform: $T_{rigid}(\mathbf{p}, \theta) = \sum_{j=1}^{N_{bones}} w_j(\mathbf{p}, \theta) \mathbf{T}_j(\theta) \mathbf{p}$, where $w_j$ are skinning weights from SMPL-X and $\mathbf{T}_j$ are bone transformation matrices.
*   Deformation: $\mathbf{p}' = T_{rigid}(\mathbf{p}, \theta) + f_{\text{deform}}(\mathbf{p}, \theta, \mathbf{w})$.
*   Covariance Update: $\Sigma' = J \Sigma J^T + \sigma^2 I$, where $J$ is the Jacobian of the deformation map with respect to $\mathbf{p}$ [G].

**Parameters:**
*   Deformation network depth/width: Not given in the sources.
*   Learning rate: Not given in the sources.

**Edge Cases:**
*   **Precision:** Deformation network must handle extreme poses without producing NaNs or exploding gradients [G].
*   **Boundaries:** Gaussians near joint boundaries may suffer from LBS artifacts; the deformation network must compensate for these [S1, unofficial].

## Subtopics

### GSMs
GSMs is a variant that generates animatable 3D humans using a model prior (SMPL) and generative adversarial networks [S2]. It builds multi-layer shells from the SMPL template and binds 3D Gaussians on these shells [S2]. It utilizes the differentiable rendering of 3DGS and the generative adversarial network of StyleGAN2 to efficiently generate animatable 3D humans [S2].

## Operating characteristics
*   **Time Complexity:** Training time depends on the complexity of the deformation network and the number of Gaussians. Rendering is $O(N)$ per pixel, similar to standard 3DGS.
*   **Memory Complexity:** $O(N)$ for Gaussian attributes plus $O(P)$ for deformation network parameters, where $N$ is the number of Gaussians and $P$ is the network size.
*   **Performance:** High fidelity for human avatars. Performance degrades if the deformation network is underfit or overfit [S1, unofficial].
*   **Strengths:** Explicit deformation allows for intuitive control and efficient inference. Compatible with standard SMPL-X rigs.
*   **Limitations:** Requires accurate SMPL-X fitting. May struggle with complex non-rigid deformations not captured by the network [S1, unofficial].
*   **When to use:** When explicit control over human animation is required and SMPL-X compatibility is needed.
*   **When not to use:** For non-human subjects or when implicit deformation is preferred for simplicity.

## Background
Animatable Gaussians is part of the evolution of 3D Gaussian Splatting for avatar reconstruction. It builds on [SMPL-based rigid/non-rigid deformation](8385-smpl-based-rigid-non-rigid-deformation.md) by introducing explicit pose-conditioned offsets [S1, unofficial]. It is cited as a key method in the landscape of Gaussian Splatting variants [S1, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
