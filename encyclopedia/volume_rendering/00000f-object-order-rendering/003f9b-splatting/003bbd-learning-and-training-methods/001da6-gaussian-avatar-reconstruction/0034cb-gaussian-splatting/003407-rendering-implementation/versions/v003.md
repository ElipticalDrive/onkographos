# Rendering Implementation

Also called: Gaussian Splatting Rasterization Pipeline, 3DGS Rendering

## Overview
Rendering Implementation defines the complete pipeline for projecting a collection of 3D Gaussians onto a 2D image plane and compositing them into a final color buffer. It solves the problem of efficiently evaluating the volume rendering equation for continuous volumetric representations by converting each primitive into a 2D splat, sorting them by depth, and accumulating color via alpha compositing [S4, S5, unofficial]. This family encompasses hardware-accelerated approaches (CPU-sorted instanced rendering, GPU tile-based rasterization) and software-based implementations, all sharing the core mathematical steps of covariance projection and alpha blending.

## Shared principles
All methods in this family rely on the following shared mechanism, terms, and math:

1.  **Primitive Representation**: Each Gaussian $i$ is defined by mean position $\mu \in \mathbb{R}^3$, scale $s \in \mathbb{R}^3$, rotation $q \in \mathbb{R}^4$ (quaternion), opacity $\alpha \in \mathbb{R}$, and spherical harmonics (SH) coefficients $c_{\ell m}$ for view-dependent color [S1][S4, unofficial].
2.  **Covariance Construction**: The 3D covariance matrix $\Sigma_{3D}$ is constructed from the rotation matrix $R$ (derived from $q$) and diagonal scaling matrix $S = \text{diag}(s)$:
    $$ \Sigma_{3D} = R S S^T R^T $$
    [S1][S3, unofficial]
3.  **Projection**: The mean is transformed to camera space $\mu' = V \mu$. The covariance is transformed to camera space $\Sigma' = V_{1:3, 1:3} \Sigma_{3D} V_{1:3, 1:3}^T$. The 2D screen-space covariance $\Sigma_{2D}$ is computed using the Jacobian $J$ of the projective projection:
    $$ \Sigma_{2D} = J \Sigma' J^T $$
    where $J = \begin{pmatrix} \frac{f_x}{\mu'_z} & 0 & -\frac{f_x \mu'_x}{\mu'_z^2} \\ 0 & \frac{f_y}{\mu'_z} & -\frac{f_y \mu'_y}{\mu'_z^2} \end{pmatrix}$ [S2, S3, unofficial].
4.  **Conic Representation**: The inverse 2D covariance $\Sigma_{2D}^{-1}$ is stored as a conic vector $C = [\Sigma_{2D}^{-1}[0,0], \Sigma_{2D}^{-1}[0,1], \Sigma_{2D}^{-1}[1,1]]$ to avoid repeated matrix inversions during rasterization [S2, S3, unofficial].
5.  **Filtering and Sorting**: Gaussians are culled if they are behind the near plane ($\mu'_z < z_{near}$) or outside the view frustum. The remaining Gaussians are sorted by camera-space depth $z$ to ensure correct front-to-back compositing [S3, S4, unofficial].
6.  **Alpha Compositing**: For each pixel, the accumulated color $C_{acc}$ and transmittance $T$ are updated for each Gaussian $i$ in depth order:
    $$ \alpha_{pixel} = \alpha_i \cdot \exp\left(-\frac{1}{2} \mathbf{x}^T \Sigma_{2D}^{-1} \mathbf{x}\right) $$
    $$ C_{acc} \leftarrow C_{acc} + T \cdot \alpha_{pixel} \cdot c_i $$
    $$ T \leftarrow T \cdot (1 - \alpha_{pixel}) $$
    where $\mathbf{x}$ is the vector from the pixel center to the Gaussian mean [S3, unofficial]. The final color includes the background: $C_{final} = C_{acc} + T \cdot C_{bg}$ [S3, unofficial].

## Subtopics

### [3D Gaussian Splatting Rendering (CPU-Sorted, Instanced)](2058-3d-gaussian-splatting-rendering-cpu-sorted-instanced.md)
This method renders scenes by projecting Gaussians into 2D screen-space quads, sorting them by camera-space depth on the CPU, and drawing them using hardware-instanced rendering with standard alpha blending. It differs from GPU-based tiled rasterization by relying on a single instanced draw call per batch, making it suitable for environments with limited GPU compute capabilities (e.g., older WebGL). It avoids per-pixel accumulation in favor of explicit back-to-front compositing via sorted instances.

### [3D Gaussian Splatting Rendering (Software)](2805-3d-gaussian-splatting-rendering-software.md)
This is a software-based rasterization method that projects 3D Gaussians to screen space using a local affine approximation, then rasterizes the resulting 2D ellipses via tile-based alpha compositing. It differs from educational MVP implementations by including the complete pipeline (data loading, projection, sorting, compositing). It uses point-sampling rather than analytical integration.

### [3D Gaussian Splatting Rasterization](2812-3d-gaussian-splatting-rasterization.md)
This is a differentiable rendering technique that projects anisotropic 3D Gaussians onto a 2D image plane for real-time rendering. It solves the problem of efficiently rendering continuous volumetric representations by converting each primitive into a 2D splat, sorting them by depth, and accumulating color via alpha compositing. It builds on [Primitive Representation](11838-primitive-representation.md) and [Projection Geometry and Coordinate Systems](9313-projection-geometry-and-coordinate-systems.md).

## Background
The rendering implementation for 3D Gaussian Splatting was introduced in the original 3DGS paper [S1]. It builds on the concept of volume splatting for efficient rasterization [Parent]. The specific optimizations, such as conic representation and tile-based rasterization, were detailed in subsequent technical reports and open-source implementations [S2, S3, unofficial].

## Sources
[S1] Kerbl, B., Kopanas, G., Leimkühler, T., & Drettakis, G. (2023). 3D Gaussian Splatting for Real-Time Radiance Field Rendering. ACM Transactions on Graphics (TOG).
[S2] 3D Gaussian Splatting GitHub Repository (lmb-informatik.uni-kiel.de).
[S3] 3D Gaussian Splatting Technical Report.
[S4] 3D Gaussian Splatting Official Website.
[S5] Unofficial Implementation Notes.
[S6] Software Rendering Implementation Details.
