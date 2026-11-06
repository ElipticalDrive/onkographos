# Camera Matrix and Coordinate Transformation

Also called: Camera-to-World Matrix, View Matrix Construction, NDC Ray Transformation

## Overview
This family defines the geometric mapping from discrete pixel coordinates to continuous 3D rays, establishing the camera's extrinsic parameters and projection model. It solves the problem of generating valid ray origins and directions for volume rendering by constructing an orthonormal basis from eye/target/up vectors and applying the appropriate projection (perspective or NDC) to map these rays into the coordinate space required by the renderer. It differs from sibling methods by focusing on the fundamental algebraic construction of the camera matrix and the coordinate transformation logic, rather than specific traversal optimizations, control interfaces, or complex distortion models.

## Shared principles
All methods in this family generate a set of rays, typically represented as origins $\mathbf{o}$ and directions $\mathbf{d}$, for every pixel in the output image [S2]. The core mechanism involves:
1. **Coordinate Mapping**: Converting discrete pixel indices $(u, v)$ or normalized device coordinates (NDC) into continuous 2D positions on a virtual image plane [S2].
2. **Camera Basis Construction**: Defining a local coordinate system for the camera using an eye position $\mathbf{e}$, a target point $\mathbf{t}$, and an up vector $\mathbf{up}$ [S3, unofficial]. This yields the basis vectors: forward $\mathbf{f} = \text{normalize}(\mathbf{t} - \mathbf{e})$, right $\mathbf{r} = \text{normalize}(\text{cross}(\mathbf{f}, \mathbf{up}))$, and camera-up $\mathbf{u} = \text{cross}(\mathbf{r}, \mathbf{f})$ [S3, unofficial].
3. **Ray Assembly**: Computing the origin $\mathbf{o}$ (often the eye position) and the direction $\mathbf{d}$ (normalized vector from origin to image plane point) [S1].
4. **Parameterization**: Sampling depth along the ray, often using linear sampling in NDC space for disparity or metric depth [S1].

## Subtopics

### [NDC Ray Transformation](905-ndc-ray-transformation.md)
Maps camera-space rays into a bounded Normalized Device Coordinate (NDC) space, typically the cube $[-1, 1]^3$, enabling the rendering of forward-facing real-world scenes that extend to infinity within a fixed bounding volume. Unlike standard perspective ray generation which samples metric depth, this method converts perspective rays into parallel rays in NDC space and uses disparity (inverse depth) as the sampling parameter, allowing linear sampling in NDC space to correspond to linear sampling in disparity from the near plane to infinity.

### [View Matrix Construction](1644-view-matrix-construction.md)
Builds a $4 \times 4$ transformation matrix that maps coordinates from world space to camera (view) space, or conversely transforms rays from view space to world space, given an eye position, a target (look-at) point, and an up vector. It solves the problem of defining a camera's extrinsic parameters in a 3D scene by establishing an orthonormal basis aligned with the viewing direction. It differs from [NDC Ray Transformation](905-ndc-ray-transformation.md) by handling only the rigid body rotation and translation of the camera, not the projection into screen coordinates, and from [Point to Camera Transformation](1660-point-to-camera-transformation.md) by explicitly constructing the full basis vectors from geometric constraints rather than applying a pre-defined transform.

### [Point to Camera Transformation](1660-point-to-camera-transformation.md)
Transforms 3D points from world coordinates to camera coordinates given a c2w matrix.

### Epipolar Geometry Rectification
**Definition**: A coordinate transformation that maps 3D points to $(\alpha, \beta, \gamma)$ where $\alpha$ specifies the epipolar slice, $\beta$ is the angle of the view ray relative to the light direction, and $\gamma$ is the distance from the light ray through the eye point. This allows view rays and light rays to be indexed independently along orthogonal axes [S1].

**Step**: To handle the epipole singularity, render results within a small ring (about two degrees) around the epipole using a standard ray marcher, and apply the epipolar algorithm only to the remainder of the image [S1].

**Pitfall**: At the epipole (where view rays and light rays are parallel), the rectification mapping is singular and the derivative $ds/d\gamma$ goes to infinity. This causes large values that disrupt the Singular Value Decomposition (SVD) [S1].

**Complexity**: Rectification of the camera depth map and unrectification take $O(sr)$ time, and rectification of the shadow map takes $O(sd)$ time, where $s$ is the number of slices, $r$ is the average number of camera rays per slice, and $d$ is the number of light rays [S1].

### Gaussian Transformation to World Space
**Definition**: Transforms Gaussian parameters predicted in camera space into world space for rendering, using a 4x4 camera-to-world extrinsics matrix $\mathbf{T}$ containing rotation $\mathbf{R}$ and translation $\mathbf{t}$ [S2].

**Formula**: The world-space mean is transformed as $\mu_{\text{world}} = \mathbf{T}\mu_{\text{cam}}$, applying the full 4x4 transformation matrix [S2]. The world-space covariance is transformed as $\Sigma_{\text{world}} = \mathbf{R}\Sigma_{\text{cam}}\mathbf{R}^T$, using only the 3x3 rotation component $\mathbf{R}$ [S2]. The world-space opacity is unchanged: $\alpha_{\text{world}} = \alpha_{\text{cam}}$ [S2]. The world-space scale is transformed as $\mathbf{s}_{\text{world}} = \mathbf{D}_R\mathbf{R}\mathbf{s}_{\text{cam}}$, where $\mathbf{D}_R$ is a block-diagonal matrix of Wigner D matrices that rotates spherical harmonics coefficients [S2].

**Parameter**: The library e3nn is used to compute the Wigner D matrices ($\mathbf{D}_R$) and to recompile the original 3D Gaussian splatting code base to follow e3nn's conventions [S2].

## Background
The construction of camera matrices from eye, center, and up vectors is a standard technique in computer graphics, often referred to as the "LookAt" function, found in libraries such as OpenGL and DirectX [S3, S9, unofficial]. The specific formulation of NDC ray transformation for forward-facing scenes, utilizing disparity sampling to handle infinite depth, is detailed in the context of neural radiance fields (NeRF) and related volume rendering techniques [S1]. The transformation of Gaussian parameters for 3D Gaussian Splatting into world space, including the use of Wigner D matrices for spherical harmonics, is a more recent development in differentiable rendering [S2].

## Sources
[S1] A hierarchical volumetric shadow algorithm for single scattering, https://doi.org/10.1145/1866158.1866200, fetched 2026-11-03
[S2] PixelSplat: 3D Gaussian Splats from Image Pairs for Scalable Generalizable 3D Reconstruction, https://doi.org/10.1109/cvpr52733.2024.01840, fetched 2026-10-26
