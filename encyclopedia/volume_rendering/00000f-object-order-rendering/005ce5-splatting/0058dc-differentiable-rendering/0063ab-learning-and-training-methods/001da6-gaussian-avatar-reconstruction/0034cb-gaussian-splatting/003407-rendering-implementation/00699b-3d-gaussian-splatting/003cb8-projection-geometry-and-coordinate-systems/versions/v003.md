# Projection Geometry and Coordinate Systems

## Overview
This family defines the mathematical framework for transforming 3D Gaussian primitives, defined by mean $\boldsymbol{\mu}$ and covariance $\boldsymbol{\Sigma}$, into 2D distributions suitable for rasterization [G]. It sets apart from standard perspective projection by supporting optimal tangent-plane projections [3293-optimal-gaussian-splatting.md] and specialized geometries such as cone-beam X-ray scanning [3458-3d-gaussian-splatting-rasterization-cone-beam-scanner-space.md].
## Shared principles
All methods in this family map a 3D Gaussian primitive to a 2D Gaussian distribution for rasterization [G]. The shared pipeline involves:
1. **World-to-Local Transformation:** Transforming the mean $\boldsymbol{\mu}$ and covariance $\boldsymbol{\Sigma}$ from world space to a local coordinate system where the viewing direction aligns with a principal axis (typically the $z$-axis) [G].
2. **Projection:** Mapping the 3D geometry to a 2D plane using a projection function $\varphi$ [S1].
3. **Covariance Handling:** Computing the 2D covariance matrix $\boldsymbol{\Sigma}_{2D}$ from the 3D covariance $\boldsymbol{\Sigma}'$ in the local space using the Jacobian $\mathbf{J}$ of the projection function: $\boldsymbol{\Sigma}_{2D} \approx \mathbf{J} \boldsymbol{\Sigma}' \mathbf{J}^\top$ [G].
4. **Rasterization:** Evaluating the 2D Gaussian at pixel coordinates to determine opacity or density contribution [G].
## Subtopics

### [Optimal Gaussian Splatting](3293-optimal-gaussian-splatting.md)
Reduces projection error by radially projecting each 3D Gaussian onto a unique tangent plane perpendicular to the line connecting the Gaussian mean to the camera center, rather than a single fixed plane $z=1$. Choose this when high-fidelity rendering of large or off-center Gaussians is required.

### [3D Gaussian Splatting Rasterization (Cone-Beam/Scanner Space)](3458-3d-gaussian-splatting-rasterization-cone-beam-scanner-space.md)
Renders X-ray projections by integrating 3D Gaussians along cone-beam ray paths, summing 2D Gaussians instead of alpha-compositing. Choose this for medical imaging or CT reconstruction tasks where integration bias must be avoided.

### Panorama Camera Adaptation
Adapts the rasterization projection for panoramic images by mapping pixel coordinates to the unit sphere [S1].
*   **Definition:** For a pixel $(u, v)$ on a panoramic image of height $H$ and width $W$, a ray is cast and mapped to the 2D coordinate vector $\mathbf{x}_{2D}$ [S1].
*   **Formula:** The projection maps to the unit sphere before applying the optimal projection function $\varphi_p$ [S1]:
    $$ \mathbf{x}_{2D} = \varphi_p \left( \begin{bmatrix} \sin \left( \frac{\pi(-W+2u)}{W} \right) \cos \left( \frac{\pi(-\frac{H}{2}+v)}{H} \right) \\ \sin \left( \frac{\pi(-\frac{H}{2}+v)}{H} \right) \\ \cos \left( \frac{\pi(-\frac{H}{2}+v)}{H} \right) \cos \left( \frac{\pi(-W+2u)}{W} \right) \end{bmatrix} \right) $$
*   **Parameters:** $H$ is the image height, $W$ is the image width [S1].

## Background
The standard 3D Gaussian Splatting method uses a fixed perspective projection plane [G]. Research has extended the projection model to support fisheye cameras, such as the equidistant projection model, and panoramic images [S1].

## Sources
[S1] On the Error Analysis of 3D Gaussian Splatting and an Optimal Projection Strategy, https://doi.org/10.1007/978-3-031-72643-9_15, fetched 2026-10-29
