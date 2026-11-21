# Deformable 3D Gaussian Splatting

Also called: Deformable 3D Gaussians, Deformable3DGS (Yang et al.), Yang et al

## Overview
Deformable 3D Gaussian Splatting is a method for dynamic scene reconstruction that learns 3D Gaussians in a canonical (static) space and models spatial-temporal deformation using an implicit deformation field implemented as a Multi-Layer Perceptron (MLP) [S1, S2]. It sets itself apart from static 3D Gaussian Splatting by enabling the rendering of time-varying scenes, and from continuous implicit methods like dynamic NeRF by maintaining the discrete, rasterizable nature of Gaussians, which makes it more sensitive to pose inaccuracies during time interpolation [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera poses, image observations at various timestamps $t$, and initial 3D Gaussian parameters in a canonical space.
*   **Outputs:** Rendered images for arbitrary timestamps and camera views.

**Data Structures:**
The method extends the Gaussian attributes defined in [Gaussian Geometry Parameters Prediction](8442-gaussian-geometry-parameters-prediction.md). Each Gaussian $i$ in the canonical space stores:
*   Position $\mathbf{p}_0 \in \mathbb{R}^3$
*   Rotation $\mathbf{q}_0 \in \mathbb{R}^4$ (unit quaternion)
*   Scaling $\mathbf{s}_0 \in \mathbb{R}^3$
*   Other attributes (opacity, spherical harmonics coefficients) which are typically treated as static or handled separately.

The deformation field is an MLP $f_{\theta}(\cdot)$ with parameters $\theta$.

**Algorithm Steps:**

1.  **Canonical Initialization:** Initialize a set of 3D Gaussians in a canonical static configuration, typically using the static 3DGS initialization pipeline [Gaussian Splatting Initialization and Parameterization](8442-gaussian-geometry-parameters-prediction.md).

2.  **Deformation Field Definition:** Define an MLP $f_{\theta}$ that takes positional encoding of the canonical position and a time encoding of the timestamp as input, and outputs offsets for position, rotation, and scaling.
    *   Input: $[\text{PE}(\mathbf{p}_0), \text{PE}(t)]$
    *   Output: $\Delta \mathbf{p}, \Delta \mathbf{q}, \Delta \mathbf{s}$

3.  **Temporal Smoothing (Noise Injection):** To improve temporal smoothing without additional computational overhead, a linearly decaying Gaussian noise is added to the encoded time vector before it enters the MLP [S2].
    *   Let $t_{enc}$ be the encoded time.
    *   $t'_{enc} = t_{enc} + \mathcal{N}(0, \sigma(t))$, where $\sigma(t)$ decays linearly with time or is fixed based on implementation details not fully specified in the summary notes [G]. *Note: The source [S2] states "linearly decaying Gaussian noise... to the encoded time vector". The exact decay function is not provided in the snippets.*

4.  **Deformation Prediction:** For a Gaussian $i$ at time $t$:
    *   Compute offsets: $[\Delta \mathbf{p}_i, \Delta \mathbf{q}_i, \Delta \mathbf{s}_i] = f_{\theta}(\text{PE}(\mathbf{p}_{0,i}), \text{PE}(t))$
    *   Apply offsets to canonical parameters to get world-space parameters at time $t$:
        *   $\mathbf{p}_i(t) = \mathbf{p}_{0,i} + \Delta \mathbf{p}_i$
        *   $\mathbf{q}_i(t) = \text{Normalize}(\mathbf{q}_{0,i} + \Delta \mathbf{q}_i)$ [G] (Standard quaternion normalization is required after addition to maintain unit length, though [S2] mentions GauFRe does this explicitly, Yang et al. implies similar handling for stability).
        *   $\mathbf{s}_i(t) = \mathbf{s}_{0,i} + \Delta \mathbf{s}_i$

5.  **Rendering:** Project the deformed Gaussians $\{\mathbf{p}_i(t), \mathbf{q}_i(t), \mathbf{s}_i(t)\}$ to 2D and rasterize using standard 3DGS splatting for the current camera view.

6.  **Optimization:** Minimize the difference between rendered and ground truth images using photometric loss. The MLP parameters $\theta$ and canonical Gaussian parameters are optimized jointly or in stages (static first, then deformation) [S2].

**Pitfalls:**
*   **Pose Sensitivity:** Inaccurate camera poses affect rendering quality significantly more than in continuous NeRF models. This is amplified in discrete 3DGS, especially during time interpolation tasks [S2].

## Subtopics

### [Shape-Shifting Splats](10708-shape-shifting-splats.md)
A method that predicts every parameter of 3D Gaussians via shallow, fully-connected MLPs conditioned on a control parameter, allowing all attributes to be changed at render time.

## Operating characteristics
*   **Performance:** On the D-NeRF dataset, Yang et al. achieves PSNR 39.51, SSIM 0.990, and LPIPS 0.012 [S2].
*   **Strengths:** Implicit deformation via MLP allows flexible modeling of complex non-rigid motions without explicit skeletal or mesh constraints.
*   **Limitations:** High sensitivity to camera pose accuracy. The use of MLPs for deformation can be computationally heavier per Gaussian than grid-based or voxel-based deformation methods (e.g., GaGS, 4D-GS) if not optimized, though the noise injection technique helps smoothing without extra overhead [S2].

## Background
Deformable 3D Gaussian Splatting was proposed by Yang et al. [19] as an extension of 3D Gaussian Splatting to dynamic scenes. It follows the paradigm of using a canonical space with an implicit deformation field, similar to dynamic NeRF [66], but adapts it to the discrete Gaussian representation [S1, S2]. It is cited as the first method to propose deformable 3D Gaussians tailored for dynamic scenes [S1].

## Sources
[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
