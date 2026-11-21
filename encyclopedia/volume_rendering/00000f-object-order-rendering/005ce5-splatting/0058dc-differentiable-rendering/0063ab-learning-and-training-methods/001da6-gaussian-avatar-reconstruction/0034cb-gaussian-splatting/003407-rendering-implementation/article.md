# Rendering Implementation

Also called: 3D Gaussian Splatting Rasterization Pipeline, 3DGS Rendering

## Overview
[G] Rendering Implementation defines the computational pipeline that transforms a set of 3D Gaussian primitives into a 2D image. [G] It solves the problem of efficiently evaluating the volume rendering equation for explicit, particle-based scene representations by leveraging hardware-accelerated rasterization. [G] It sets itself apart from its siblings by providing the foundational mechanisms—specifically the projection from 3D covariance to 2D screen-space conics and the alpha compositing logic—that enable both the high-performance GPU rasterization of [3D Gaussian Splatting](2812-3d-gaussian-splatting.md) and the display-only CPU-sorted instancing of [3D Gaussian Splatting Rendering (CPU-Sorted, Instanced)](2058-3d-gaussian-splatting-rendering-cpu-sorted-instanced.md).

## Shared principles
All rendering implementations for 3D Gaussians share the following mathematical and procedural core, as defined in [Gaussian Splatting](946-gaussian-splatting.md):

1. **Parameter Activation**: Raw training parameters are transformed into geometric primitives:
    * Rotation $q$ is normalized: $q_{\text{norm}} = q / \|q\|$ [Gaussian Splatting](946-gaussian-splatting.md).
    * Scale $s$ is exponentiated to ensure positivity: $s_{\text{eff}} = \exp(s)$ [Gaussian Splatting](946-gaussian-splatting.md).
    * Opacity $\alpha$ is sigmoided: $\alpha_{\text{eff}} = \sigma(\alpha)$ [Gaussian Splatting](946-gaussian-splatting.md).
    * The 3D covariance matrix is constructed as $\boldsymbol{\Sigma}_{3D} = \mathbf{R} \mathbf{S} \mathbf{S}^T \mathbf{R}^T$, where $\mathbf{R}$ is the rotation matrix derived from $q_{\text{norm}}$ and $\mathbf{S} = \text{diag}(s_{\text{eff}})$ [Gaussian Splatting](946-gaussian-splatting.md).

2. **Camera Space Transformation**:
    * The mean position is transformed to camera space: $\boldsymbol{\mu}' = \mathbf{V} \boldsymbol{\mu}$ [Gaussian Splatting](946-gaussian-splatting.md).
    * The covariance is transformed using the upper-left $3 \times 3$ submatrix of the view matrix $\mathbf{V}$:
        $$ \boldsymbol{\Sigma}' = \mathbf{V}_{1:3, 1:3} \boldsymbol{\Sigma}_{3D} \mathbf{V}_{1:3, 1:3}^T $$
        [Gaussian Splatting](946-gaussian-splatting.md).

3. **Screen-Space Projection**:
    * The Jacobian $\mathbf{J}$ of the projective projection at $\boldsymbol{\mu}'$ is computed:
        $$ \mathbf{J} = \begin{pmatrix} \frac{f_x}{\mu'_z} & 0 & -\frac{f_x \mu'_x}{\mu'_z^2} \\ 0 & \frac{f_y}{\mu'_z} & -\frac{f_y \mu'_y}{\mu'_z^2} \end{pmatrix} $$
        [Gaussian Splatting](946-gaussian-splatting.md).
    * The 2D screen-space covariance is:
        $$ \boldsymbol{\Sigma}_{2D} = \mathbf{J} \boldsymbol{\Sigma}' \mathbf{J}^T $$
        [Gaussian Splatting](946-gaussian-splatting.md).
    * The 2D mean is:
        $$ \boldsymbol{\mu}_{2D} = \begin{pmatrix} \frac{f_x \mu'_x}{\mu'_z} + c_x \\ \frac{f_y \mu'_y}{\mu'_z} + c_y \end{pmatrix} $$
        [Gaussian Splatting](946-gaussian-splatting.md).

4. **Alpha Compositing**:
    * Gaussians are sorted by depth (camera-space $z$) [Gaussian Splatting](946-gaussian-splatting.md).
    * For each pixel, color $C$ and transmittance $T$ are accumulated:
        $$ C \leftarrow C + T \cdot \alpha_{\text{pixel}} \cdot \mathbf{c} $$
        $$ T \leftarrow T \cdot (1 - \alpha_{\text{pixel}}) $$
        [Gaussian Splatting](946-gaussian-splatting.md).
    * Early termination occurs when $T$ drops below a threshold (e.g., 0.05) [Gaussian Splatting](946-gaussian-splatting.md).

## Subtopics

### [3D Gaussian Splatting Rendering (CPU-Sorted, Instanced)](2058-3d-gaussian-splatting-rendering-cpu-sorted-instanced.md)
A display-only technique that projects Gaussians to 2D quads, sorts them by depth on the CPU, and renders them via hardware-instanced drawing with standard alpha blending. It avoids GPU tile-based complexity by relying on CPU sorting.

### [3D Gaussian Splatting](2812-3d-gaussian-splatting.md)
The primary real-time rendering method using GPU-based global depth sorting and tile-based rasterization. It computes 2D conics for efficient pixel-wise alpha evaluation and supports view-dependent color via spherical harmonics.

## Background
[G] The rendering pipeline originates from the 3D Gaussian Splatting (3DGS) framework, which introduced explicit Gaussian primitives to replace volumetric ray marching for real-time novel view synthesis. [G] The method was developed to overcome the computational cost of NeRFs while maintaining high visual fidelity. [G] The specific projection and rasterization techniques described here are standard implementations of the 3DGS rendering equation.

## Sources
[S1] Kerbl, B., Kopanas, G., Leimkühler, T., & Drettakis, G. (2023). 3D Gaussian Splatting for Real-Time Radiance Field Rendering. ACM Transactions on Graphics (TOG).
[S2] Unofficial implementation details from various open-source 3DGS renderers (e.g., InstantNGP, Gaussian Splatting CUDA kernels).
[S6] Unofficial community documentation and source code analysis of the official 3DGS repository.
[S8] Unofficial notes on Spherical Harmonics evaluation in 3DGS.
[S9] Unofficial notes on 3D covariance construction.
[S12] Unofficial notes on tile-based rasterization parameters.
[S13] Unofficial notes on output formats.
[S14] Unofficial notes on data structures and culling.
[S16] Unofficial notes on GPU sorting algorithms (CUB).
[S18] Unofficial comparison with NeRF.
[S19] Unofficial notes on Jacobian approximation artifacts.
[S23] Unofficial notes on initialization strategies.
