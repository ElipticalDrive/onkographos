# Camera and Projection Models

Also called: Camera Models, Projection Models

## Overview
Camera and Projection Models define the geometric transformation from 3D scene coordinates to 2D image coordinates, serving as the foundation for differentiable rendering in Gaussian Splatting [G]. This family replaces or augments the standard perspective projection of 3D Gaussian Splatting to support specific optical characteristics, such as defocus blur or fisheye distortion, or to optimize computational efficiency for specific domains like satellite imagery [G]. It sets itself apart from standard 3DGS by modifying the projection function $\phi$ and its Jacobian, thereby altering how 2D covariances are derived and how depth-dependent effects (like Circle of Confusion) are modeled [G].

## Shared principles
All methods in this family share the core mechanism of mapping 3D Gaussian primitives $\mathcal{S}$ to 2D screen space for rasterization [G]. The fundamental steps are:
1.  **View Transformation:** Transform the 3D mean $\boldsymbol{\mu}$ and covariance $\boldsymbol{\Sigma}$ from world coordinates to camera coordinates using the viewing matrix $\mathbf{W}$ and translation $\mathbf{b}$:
    $$ \boldsymbol{\mu}_c = \mathbf{W}\boldsymbol{\mu} + \mathbf{b} $$
2.  **Projection:** Apply a projection function $\phi: \mathbb{R}^3 \to \mathbb{R}^2$ to map the camera-space mean to 2D pixel coordinates:
    $$ \boldsymbol{\mu}_p = \phi(\boldsymbol{\mu}_c) $$
3.  **Covariance Projection:** Compute the 2D covariance $\boldsymbol{\Sigma}'$ using the Jacobian $\mathbf{J}$ of the projection function $\phi$ with respect to the camera-space coordinates, and the viewing rotation $\mathbf{W}$:
    $$ \boldsymbol{\Sigma}' = \mathbf{J} \mathbf{W} \boldsymbol{\Sigma} \mathbf{W}^T \mathbf{J}^T $$
    This affine approximation ensures the projected Gaussian remains a valid 2D Gaussian primitive [S1].

## Subtopics

### [Differentiable DOF Rendering](18289-differentiable-dof-rendering.md)
Differentiable DOF Rendering extends the projection model to simulate depth-of-field effects by convolving projected 2D Gaussians with depth-dependent blur kernels [18289-differentiable-dof-rendering.md]. It sets itself apart by treating the Circle of Confusion (CoC) radius as a differentiable parameter derived from camera aperture and focal distance, allowing joint optimization of scene geometry and camera intrinsics. It is chosen when realistic optical blur is required for training or rendering.

### [Fisheye-GS](20662-fisheye-gs.md)
Fisheye-GS replaces the standard perspective projection with an equidistant fisheye model to handle extreme curvature and wide fields of view [20662-fisheye-gs.md]. It sets itself apart by recalculating the projection transformation and optimizing tile-based rasterization to reduce computational redundancies caused by high splat density in wide-angle views. It is chosen for scenes captured with fisheye lenses.

### Affine Camera Approximation
#### definition
- [S1] A per-scene affine approximation of the transformation from world-space to 2D NDC-space, replacing the complex RPC-based mapping to improve computational efficiency while maintaining compatibility with Gaussian Splatting.
#### formula
- [S1] $\mu^A_k = A\mu_k + a \in \mathbb{R}^2$ and $\Sigma^A_{2\times2} = A\Sigma A^T \in \mathbb{R}^{2\times2}$, where $A: \mathbb{R}^3 \to \mathbb{R}^2$ is the affine camera model, $\mu_k$ is the Gaussian mean, $a$ is the translation vector, $\Sigma$ is the covariance matrix, and $A^T$ is the transpose of $A$.
#### parameter
- [S1] The affine model $A$ maps 3D world coordinates to 2D NDC coordinates normalized to range [-1, 1].
#### pitfall
- [S1] The original 3DGS method uses a local first-order approximation for projection; this method eliminates that need by moving the approximation to the camera model definition itself.
#### number
- [S1] Mean error of the affine approximation is approximately 0.012 pixels compared to the full RPC transformation.

## Background
The standard perspective projection model was introduced in the original 3D Gaussian Splatting paper [G]. The affine camera approximation for satellite imagery was introduced in "Gaussian Splatting for Efficient Satellite Image Photogrammetry" [S1] to handle Rational Polynomial Coefficient (RPC) models efficiently. Differentiable DOF rendering and Fisheye-GS are extensions developed to handle specific optical phenomena not covered by the standard pinhole camera model [G].

## Sources
[S1] Gaussian Splatting for Efficient Satellite Image Photogrammetry, https://doi.org/10.1109/cvpr52734.2025.00559, fetched 2026-10-26
