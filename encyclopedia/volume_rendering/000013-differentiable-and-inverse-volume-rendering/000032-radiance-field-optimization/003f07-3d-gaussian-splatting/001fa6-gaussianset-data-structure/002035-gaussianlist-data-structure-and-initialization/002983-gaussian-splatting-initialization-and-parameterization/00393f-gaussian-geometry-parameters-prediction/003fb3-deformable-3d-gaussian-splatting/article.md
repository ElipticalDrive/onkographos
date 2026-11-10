# Deformable 3D Gaussian Splatting

Also called: Deformable 3D Gaussians, Deformable3DGS (Yang et al.), Yang et al

## Overview
Deformable 3D Gaussian Splatting is a method for dynamic scene reconstruction that learns 3D Gaussians in a canonical (static) space and models spatial-temporal deformation using an implicit deformation field implemented as a Multi-Layer Perceptron (MLP) [S1, S2]. It differs from static 3D Gaussian Splatting by introducing time-dependent offsets to Gaussian attributes, and from other dynamic variants like [Mesh-Embedded Gaussian Splatting](9553-mesh-embedded-gaussian-splatting.md) by relying on an implicit MLP rather than explicit mesh constraints or voxel grids [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Canonical 3D Gaussian parameters (position $\mathbf{p}_0$, rotation $\mathbf{q}_0$, scale $\mathbf{s}_0$, opacity $\alpha_0$) stored in a canonical space; a camera pose; a timestep $t$.
*   **Outputs:** Deformed Gaussian parameters ($\mathbf{p}, \mathbf{q}, \mathbf{s}$) at time $t$, and rendered image.

**Data Structures:**
*   **Canonical Gaussians:** Stored as a list of Gaussians. Each Gaussian has fields for position $\mathbf{p}_0 \in \mathbb{R}^3$, rotation quaternion $\mathbf{q}_0 \in \mathbb{R}^4$, scale vector $\mathbf{s}_0 \in \mathbb{R}^3$, and opacity $\alpha_0 \in \mathbb{R}$.
*   **Deformation MLP:** A neural network that takes positional-encoded position and time as input and outputs offsets.

**Steps:**
1.  **Canonical Initialization:** Initialize a set of 3D Gaussians in a canonical static space using standard 3DGS initialization methods [S2].
2.  **Time Encoding:** For a given timestep $t$, compute a positional encoding $\gamma(t)$ [S2]. To improve temporal smoothing without additional computational overhead, add linearly decaying Gaussian noise to the encoded time vector [S2].
3.  **Deformation Prediction:** For each Gaussian $i$ with canonical position $\mathbf{p}_{0,i}$, pass the concatenation of the positional encoding of $\mathbf{p}_{0,i}$ and the encoded time $t$ into the deformation MLP [S2].
4.  **Offset Application:** The MLP outputs offsets for position ($\Delta \mathbf{p}$), rotation ($\Delta \mathbf{q}$), and scaling ($\Delta \mathbf{s}$) [S2].
    *   Deformed position: $\mathbf{p}_i = \mathbf{p}_{0,i} + \Delta \mathbf{p}_i$
    *   Deformed rotation: $\mathbf{q}_i = \text{Normalize}(\mathbf{q}_{0,i} + \Delta \mathbf{q}_i)$ [G]
    *   Deformed scale: $\mathbf{s}_i = \mathbf{s}_{0,i} + \Delta \mathbf{s}_i$ [G]
5.  **Rendering:** Render the deformed Gaussians using standard differentiable rasterization [S2].

**Pitfalls:**
*   Inaccurate camera poses significantly affect rendering quality, a problem amplified in discrete 3DGS compared to continuous NeRF, especially during time interpolation [S2].

## Subtopics

### [Shape-Shifting Splats](10708-shape-shifting-splats.md)
A dynamic reconstruction method that predicts all 3D Gaussian parameters via shallow, fully-connected MLPs conditioned on a control parameter, allowing all attributes to be changed at render time. It builds on Deformable 3D Gaussian Splatting by replacing explicit storage of scale, rotation, offset, and opacity with learned functions, enabling context translation (e.g., RGB to thermal) and stylization without retraining the scene geometry.

## Operating characteristics
*   **Performance:** On the D-NeRF dataset, Yang et al. achieves PSNR 39.51, SSIM 0.990, and LPIPS 0.012 [S2].
*   **Strengths:** Implicit deformation field allows flexible modeling of complex dynamics without explicit topology [S1].
*   **Limitations:** Sensitive to inaccurate camera poses, particularly in time interpolation tasks [S2].

## Background
Deformable 3D Gaussian Splatting was proposed by Yang et al. [19] as a tailored solution for dynamic scenes, building on the idea of learning Gaussians in a canonical space and modeling deformation with an implicit MLP, similar to dynamic NeRF [S1, S2].

## Sources
[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
