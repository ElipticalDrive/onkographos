# Projection Geometry and Coordinate Systems

Also called: 3D Gaussian Splatting Projection, Gaussian Splatting Coordinate Transformations

## Overview
This article defines the mathematical framework for transforming 3D Gaussian primitives from world space into the 2D image plane for rasterization in 3D Gaussian Splatting. It solves the problem of accurately projecting 3D ellipsoids onto a 2D sensor plane while preserving geometric fidelity and enabling differentiable rendering. It sets apart from standard perspective projection by supporting optimal tangent-plane projections [3293-optimal-gaussian-splatting.md] and specialized geometries such as cone-beam X-ray scanning [3458-3d-gaussian-splatting-rasterization-cone-beam-scanner-space.md].

## Shared principles
All methods in this family share the core objective of mapping a 3D Gaussian primitive, defined by mean $\boldsymbol{\mu}$ and covariance $\boldsymbol{\Sigma}$, to a 2D Gaussian distribution suitable for alpha blending or density summation. The general pipeline involves:
1.  **World-to-Camera/Scanner Transformation:** Transforming the primitive's mean and covariance into a local coordinate system where the viewing direction is aligned with a principal axis (typically the z-axis).
2.  **Projection:** Mapping the 3D geometry to a 2D plane. This can be a standard perspective projection, an optimal tangent-plane projection, or a ray-space integration for non-standard geometries.
3.  **Covariance Handling:** Computing the 2D covariance matrix $\boldsymbol{\Sigma}_{2D}$ from the 3D covariance $\boldsymbol{\Sigma}'$ in the local space. This often involves a Jacobian matrix $\mathbf{J}$ of the projection function: $\boldsymbol{\Sigma}_{2D} \approx \mathbf{J} \boldsymbol{\Sigma}' \mathbf{J}^\top$.
4.  **Rasterization:** Evaluating the 2D Gaussian at pixel coordinates to determine opacity or density contribution.

## Subtopics

### [Optimal Gaussian Splatting](3293-optimal-gaussian-splatting.md)
Reduces projection error by radially projecting each 3D Gaussian onto a unique tangent plane perpendicular to the line connecting the Gaussian mean to the camera center, rather than a single fixed plane $z=1$. Choose this when high-fidelity rendering of large or off-center Gaussians is required.

### [3D Gaussian Splatting Rasterization (Cone-Beam/Scanner Space)](3458-3d-gaussian-splatting-rasterization-cone-beam-scanner-space.md)
Renders X-ray projections by integrating 3D Gaussians along cone-beam ray paths, summing 2D Gaussians instead of alpha-compositing. Choose this for medical imaging or CT reconstruction tasks where integration bias must be avoided.

### Panorama Camera Adaptation
Adapts the rasterization projection for panoramic images by mapping pixel coordinates to the unit sphere using equisolid angle or similar projection models.
*   **Definition:** For a pixel $(u, v)$ on a panoramic image of height $H$ and width $W$, a ray is cast and mapped to the 2D coordinate vector $\mathbf{x}_{2D}$ [S1].
*   **Formula:** The projection uses the following mapping to the unit sphere before applying the optimal projection function $\varphi_p$ [S1]:
    $$ \mathbf{x}_{2D} = \varphi_p \left( \begin{bmatrix} \sin \left( \frac{\pi(-W+2u)}{W} \right) \cos \left( \frac{\pi(-\frac{H}{2}+v)}{H} \right) \\ \sin \left( \frac{\pi(-\frac{H}{2}+v)}{H} \right) \\ \cos \left( \frac{\pi(-\frac{H}{2}+v)}{H} \right) \cos \left( \frac{\pi(-W+2u)}{W} \right) \end{bmatrix} \right) $$
*   **Parameters:** $H$ is the image height, $W$ is the image width [S1].

## Background
The standard 3D Gaussian Splatting method [17] uses a fixed perspective projection plane. Subsequent research identified errors in this approximation, particularly for Gaussians far from the principal axis, leading to the development of optimal projection strategies [S1]. The cone-beam adaptation extends the method to non-standard imaging geometries like X-ray CT [S1].

## Sources
[S1] On the Error Analysis of 3D Gaussian Splatting and an Optimal Projection Strategy, https://doi.org/10.1007/978-3-031-72643-9_15, fetched 2026-10-29
