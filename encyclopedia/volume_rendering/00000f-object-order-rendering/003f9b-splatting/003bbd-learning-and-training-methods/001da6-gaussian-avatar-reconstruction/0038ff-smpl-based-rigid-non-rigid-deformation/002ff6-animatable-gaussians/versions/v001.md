# Animatable Gaussians

## Overview
Animatable Gaussians is a method for reconstructing animatable 3D human avatars from monocular video by driving explicit 3D Gaussians with an SMPL-X rig [S1, unofficial]. It differs from sibling methods like [3DGS-Avatar](1480-3dgs-avatar.md), which uses a per-Gaussian deformation MLP, by employing explicit pose-conditioned offsets to determine Gaussian positions [S1, unofficial]. This approach provides a direct mapping between the parametric model's pose and the Gaussian geometry, avoiding the need for learned implicit deformation fields for position [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Monocular video of a human subject; pre-fitted SMPL-X parameters (pose $\beta$, shape $\theta$, and translation) for each frame [S1, unofficial].
*   **Outputs:** A set of 3D Gaussians $\{G_i\}$ where each Gaussian $i$ has a base position $\mathbf{p}_i$, covariance $\Sigma_i$, opacity $\alpha_i$, and spherical harmonic coefficients $\mathbf{c}_i$, which are deformed based on the input pose [S1, unofficial].

**Data Structures:**
Each Gaussian $i$ stores:
*   $\mathbf{p}_i \in \mathbb{R}^3$: Base position in the SMPL-X rest pose space.
*   $\Sigma_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix (often parameterized by scale $\mathbf{s}_i$ and rotation $\mathbf{q}_i$).
*   $\alpha_i \in [0, 1]$: Opacity.
*   $\mathbf{c}_i$: Spherical harmonic coefficients for view-dependent color.
*   The deformation is defined by an offset function $\Delta \mathbf{p}(\text{pose}, \text{index})$ [S1, unofficial].

**Steps:**
1.  **Initialization:** Initialize Gaussians on the SMPL-X mesh surface in the rest pose [S1, unofficial].
2.  **Deformation:** For a given frame with pose parameters $\theta$, compute the deformed position $\mathbf{p}'_i$ for each Gaussian:
    $$ \mathbf{p}'_i = \mathbf{T}(\theta) \mathbf{p}_i + \Delta \mathbf{p}_i(\theta) $$
    where $\mathbf{T}(\theta)$ is the SMPL-X skinning transform and $\Delta \mathbf{p}_i$ is the explicit pose-conditioned offset [S1, unofficial].
3.  **Rendering:** Render the deformed Gaussians using standard differentiable splatting [S1, unofficial].
4.  **Optimization:** Minimize photometric loss between rendered images and ground truth video frames to optimize Gaussian attributes and potentially refine the offset function or SMPL parameters [S1, unofficial].

**Parameters:**
*   SMPL-X rig parameters (pose, shape) [S1, unofficial].
*   Learning rate for Gaussian attributes and offsets [S1, unofficial].

**Edge Cases:**
*   Precision issues in skinning transforms for extreme poses [G].
*   Degenerate input where SMPL fitting fails [G].

## Subtopics

### GSMs
GSMs is a variant that generates animatable 3D humans using a model prior (SMPL) and generative adversarial networks [S2]. It builds multi-layer shells from the SMPL template and binds 3D Gaussians on these shells [S2]. By utilizing the differentiable rendering of 3DGS and the generative adversarial network of StyleGAN2, it efficiently generates animatable 3D humans [S2].

## Operating characteristics
*   **Time Complexity:** Depends on the number of Gaussians and the cost of computing explicit offsets. Generally faster than MLP-based deformation methods as it avoids per-pixel MLP queries for position [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of Gaussians. Storing explicit offsets may require additional memory compared to fixed deformation [S1, unofficial].
*   **Performance:** High-quality photorealistic reconstruction from monocular video [S1, unofficial].
*   **Strengths:** Explicit deformation allows for intuitive control via SMPL-X parameters [S1, unofficial].
*   **Limitations:** Quality depends on the accuracy of the SMPL-X fit and the ability of explicit offsets to capture complex non-rigid deformations [S1, unofficial].
*   **When to use:** When monocular video input is available and explicit control via SMPL-X is desired [S1, unofficial].
*   **When not to use:** When high-fidelity non-rigid deformation beyond SMPL-X capabilities is required, or when multi-view input is available for better geometry [S1, unofficial].

## Background
Animatable Gaussians was introduced as part of the evolution of Gaussian Splatting for avatar reconstruction [S1, unofficial]. It builds on the SMPL-based rigid/non-rigid deformation principles [S1, unofficial]. The method addresses the challenge of creating animatable avatars from monocular video by leveraging the explicit structure of the SMPL-X rig [S1, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
