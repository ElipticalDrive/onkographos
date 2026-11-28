# Base 3DGS Optimization

Also called: Base 3DGS Optimization with Normal Attribute

## Overview
Base 3DGS Optimization initializes the geometry attributes of 3D Gaussian primitives to provide a robust foundation for subsequent shading attribute optimization, such as in iVR-GS [S1]. It differs from standard 3DGS training by incorporating an explicit normal attribute $n$ for each Gaussian and optimizing it via a normal consistency loss, even though this attribute does not affect the base rendering output [S1]. This initialization accelerates the convergence of editable Gaussian optimization by ensuring accurate geometric representation before decomposing color and lighting [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Multi-view images, camera poses, initial 3D Gaussian points (typically from SfM).
*   **Output:** Optimized 3D Gaussian primitives with parameters $\{\mu_i, q_i, s_i, o_i, n_i, c_i\}$, where $n_i$ is the normal attribute and $c_i$ is represented by SH coefficients [S1].

**Data Structures:**
Each Gaussian $i$ is parameterized as:
*   $\mu_i \in \mathbb{R}^3$: Mean position.
*   $q_i \in \mathbb{R}^4$: Rotation quaternion.
*   $s_i \in \mathbb{R}^3$: Scale vector.
*   $o_i \in [0, 1]$: Opacity.
*   $n_i \in \mathbb{R}^3$: Normal attribute (unit vector).
*   $c_i$: SH coefficients for view-dependent color [S1].

**Numbered Steps:**
1.  **Render Depth and Normal Maps:** For a given viewpoint, render the depth map $D$ and normal map $N$ using the current Gaussian parameters. The rendering follows the alpha compositing equation [S1]:
    $$ \{D, N\} = \sum_{i \in \mathcal{N}} T_i \alpha_i \{d_i, n_i\} $$
    where $\mathcal{N}$ is the set of splatted Gaussians, $T_i$ is the transmittance, $\alpha_i$ is the opacity, $d_i$ is the depth of the $i$-th Gaussian, and $n_i$ is its normal attribute [S1]. Note that $n_i$ is not used in the standard 3DGS color/alpha accumulation; it is only accumulated into the normal map buffer for loss computation [S1].

2.  **Compute Pseudo Normal:** Calculate a pseudo normal map $\tilde{N}$ from the rendered depth map $D$ under the local planarity assumption. This is typically done by computing the cross product of the spatial gradients of the depth map [S1][S2, unofficial].

3.  **Compute Losses:** Calculate the total loss $L$ as a combination of color/alpha losses and the normal consistency loss [S1]:
    $$ L = L_{color} + L_{alpha} + L_n $$
    Where:
    *   $L_{color}$ is the L1 loss between rendered and ground truth RGB colors.
    *   $L_{alpha}$ is the L1 loss between rendered and ground truth alpha channels (optimizing alpha enhances reconstruction quality [S1]).
    *   $L_{ssim}$ is the SSIM loss between rendered and ground truth RGB images.
    *   $L_n$ is the normal consistency loss [S1]:
        $$ L_n = ||N - \tilde{N}||_2 $$

4.  **Optimize:** Update all Gaussian parameters $\{\mu_i, q_i, s_i, o_i, n_i, c_i\}$ using an optimizer (e.g., Adam) to minimize $L$. The normal attribute $n_i$ is updated despite not affecting the base rendering, to accelerate convergence for subsequent stages [S1].

**Parameters:**
*   Loss weights for $L_{color}$, $L_{alpha}$, $L_{ssim}$, and $L_n$ are not explicitly defined in the source for the base method, but standard 3DGS weights are implied for the color/alpha/SSIM components [S1].
*   The normal consistency loss weight is not specified in the base definition [S1].

**Edge Cases and Pitfalls:**
*   Adjusting the normal attribute $n$ does not affect the rendering results of base 3DGS; it is optimized solely to accelerate the convergence of editable Gaussian optimization [S1].
*   The pseudo normal $\tilde{N}$ relies on the local planarity assumption, which may fail in highly non-planar regions or where depth is noisy [S1].

## Subtopics

### [iVR-GS Training](7578-ivr-gs-training.md)
iVR-GS Training is the optimization procedure for Inverse Volume Rendering Gaussian Splatting (iVR-GS), a method that extends 3D Gaussian Splatting with editable shading attributes for volume visualization. It solves the instability and inefficiency of jointly optimizing base geometry and deformation-aware primitives by decoupling the process into two distinct stages: first establishing a robust base 3DGS model, then refining editable Gaussian primitives on top of it. This approach accelerates convergence and reduces memory footprint compared to one-stage training while maintaining reconstruction quality.

### [3D Smoothing Filter](9588-3d-smoothing-filter.md)
The 3D Smoothing Filter prevents 3D Gaussian primitives from becoming smaller than the maximal sampling frequency induced by the training images, thereby mitigating aliasing artifacts during rendering at varying sampling rates. It differs from its siblings, [iVR-GS Training](7578-ivr-gs-training.md) and [Two-stage Gaussian-Surfel (GES) Optimization](23119-two-stage-gaussian-surfel-ges-optimization.md), by explicitly constraining the Gaussian scale based on camera geometry rather than relying on separate shading attribute optimization or surfel-specific regularization.

### [Two-stage Gaussian-Surfel (GES) Optimization](23119-two-stage-gaussian-surfel-ges-optimization.md)
Two-stage Gaussian-Surfel (GES) optimization is a training pipeline for differentiable rendering that separates the initialization of opaque surface geometry from the joint optimization of appearance and fine geometry. It solves the problem of color leakage and order-independent blending artifacts that occur when translucent surfels and 3D Gaussians are optimized simultaneously. Unlike single-stage approaches where Gaussians dominate image fitting at the expense of surfel coverage, GES first establishes a coarse, opaque surfel backbone, then enriches it with 3D Gaussians for high-frequency details.

### Editable Gaussian Optimization
Editable Gaussian Optimization decomposes color and lighting from the scene using the Blinn-Phong model instead of SH coefficients, enabling scene editing [S1]. It builds on the Base 3DGS Optimization by assigning additional shading attributes to each Gaussian primitive: offset color $\Delta c \in \mathbb{R}^3$, ambient/diffuse/specular coefficients $\{k_a, k_d, k_s\}$, and shininess term $\beta$ [S1]. The voxel color $c_v$ is expressed as $c_p + \Delta c$, where $c_p \in \mathbb{R}^3$ is a palette color parameter shared by all Gaussians within a basic scene, fixed as the mean color value of input multi-view images during training, and adjustable during inference [S1]. To reduce model size, low-opacity Gaussians are removed periodically, and an L1 regularization term encourages low opacity to neutralize the tendency to use more Gaussians for compact surfaces due to normal-dependent color [S1]. Sparsity regularization (L1 loss of rendered offset color map) is added to $\Delta c$ to avoid significant palette color shiftings [S1].

### Geometry-Guided Optimization
Geometry-Guided Optimization is an optimization phase using Edge-Aware Normal Loss and Multi-Scale SSIM loss to ensure geometric accuracy without affecting RGB fidelity [S2, unofficial]. It computes a normal map $N$ from an unprojected depth map $D$ using the cross product of gradients: $N = (\nabla_x D \times \nabla_y D) / ||\nabla_x D \times \nabla_y D||$ [S2, unofficial]. It derives a curvature map $|\nabla N| \in [0, 1]^{H \times W}$ as the gradient magnitude of the normal map and an edge map $|\nabla I| \in [0, 1]^{H \times W}$ from the gradient magnitude of the ground truth RGB image [S2, unofficial]. A weight function $\omega(x) = (x - 1)^q$ (where $q$ is a positive even integer) balances the edge map's influence on the curvature map [S2, unofficial]. The Edge-Aware Normal Loss is $L_{normal} = \frac{1}{HW} \sum |\nabla N| \otimes \omega(|\nabla I|)$ [S2, unofficial]. It replaces SSIM loss with Multi-Scale SSIM (MS-SSIM) loss [S2, unofficial]. The composite loss is $L = (1 - \lambda_{ms-ssim})L_1 + \lambda_{ms-ssim} L_{ms-ssim} + \lambda_{normal} L_{normal}$ [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Accelerates convergence of subsequent shading attribute optimization by providing accurate geometric initialization [S1]. Improves reconstruction quality by optimizing alpha channels [S1].
*   **Limitations:** The normal attribute does not affect base rendering, adding computational overhead for loss computation without direct visual benefit in the base stage [S1].
*   **When to use:** When subsequent stages require accurate geometric initialization, such as in iVR-GS [S1].
*   **When not:** When only base rendering quality is required without subsequent editing or shading decomposition [S1].

## Background
Base 3DGS Optimization was introduced in the context of iVR-GS (Inverse Volume Rendering Gaussian Splatting) to enable editable visualization by decomposing color and lighting [S1]. It builds on the original 3DGS method by adding a normal attribute and optimizing it via a consistency loss with pseudo normals derived from depth [S1]. The method leverages the fact that 3DGS reconstructs per-pixel alpha channel information, which can enhance reconstruction quality when optimized [S1].

## Sources
[S1] iVR-GS: Inverse Volume Rendering for Explorable Visualization via Editable 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3567121, fetched 2026-10-26
[S2] AtomGS: Atomizing Gaussian Splatting for High-Fidelity Radiance Field, https://core.ac.uk/works/391753628, fetched 2026-10-30 (unofficial)
