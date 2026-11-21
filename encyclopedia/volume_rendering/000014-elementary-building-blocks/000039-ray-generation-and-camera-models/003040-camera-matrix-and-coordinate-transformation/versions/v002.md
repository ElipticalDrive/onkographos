# Camera Matrix and Coordinate Transformation


## Overview
This family defines the geometric mapping from discrete pixel coordinates to continuous 3D rays, establishing the camera's extrinsic parameters and projection model [G]. It solves the problem of generating valid ray origins and directions for volume rendering by constructing an orthonormal basis from eye/target/up vectors and applying the appropriate projection, differing from sibling methods by focusing on fundamental algebraic construction rather than traversal optimizations or control interfaces [G].

## Shared principles
Builds on the geometric definition of rays and the parallel execution model from [Ray generation and camera models](770-ray-generation-and-camera-models.md).

## Subtopics

### [NDC Ray Transformation](905-ndc-ray-transformation.md)
Maps camera-space rays into a bounded Normalized Device Coordinate (NDC) cube $[-1, 1]^3$ to enable uniform sampling in disparity for forward-facing scenes with unbounded depth. Unlike standard ray generation which samples metric depth linearly, this method shifts ray origins to the near plane and projects them such that linear sampling in the transformed NDC parameter corresponds to linear sampling in disparity from the near plane to infinity. This allows rendering scenes extending to infinity within a fixed bounding volume, a requirement for real-world forward-facing captures like those in the LLFF dataset. Choose for forward-facing captures requiring unbounded depth handling.

### [View Matrix Construction](1644-view-matrix-construction.md)
Generates a $4 \times 4$ transformation matrix that maps coordinates from world space to camera (view) space, or conversely transforms rays from view space to world space, given an eye position, a target point, and an up vector. It solves the problem of defining a camera's orientation and position relative to the scene by constructing an orthonormal basis aligned with the viewing direction. Unlike [NDC Ray Transformation](905-ndc-ray-transformation.md), which handles projection and depth mapping, or [Point to Camera Transformation](1660-point-to-camera-transformation.md), which may apply arbitrary rigid transforms, this method specifically derives the camera's local coordinate frame from geometric constraints. It sets itself apart by explicitly ensuring the camera looks down the negative z-axis in view space, a convention required for standard ray marching and volume rendering pipelines. Choose when defining camera orientation and position relative to scene geometry.

### [Point to Camera Transformation](1660-point-to-camera-transformation.md)
Converts 3D points from world coordinates into the camera's local coordinate system. It serves as the foundational step for ray generation and visibility checks by establishing the relative position of scene geometry with respect to the viewer. Unlike [NDC Ray Transformation](905-ndc-ray-transformation.md), which maps pixels to rays, or [View Matrix Construction](1644-view-matrix-construction.md), which builds the full affine transform matrix, this method explicitly computes the rotated and translated position of individual points to facilitate downstream operations like depth sampling or projection. Choose when establishing relative position of scene geometry for visibility checks.

### Epipolar Geometry Rectification
**Definition**: A coordinate transformation that maps 3D points to $(\alpha, \beta, \gamma)$ where $\alpha$ specifies the epipolar slice, $\beta$ is the angle of the view ray relative to the light direction, and $\gamma$ is the distance from the light ray through the eye point. This allows view rays and light rays to be indexed independently along orthogonal axes [S1].
**Step**: To handle the epipole singularity, render results within a small ring (about two degrees) around the epipole using a standard ray marcher, and apply the epipolar algorithm only to the remainder of the image [S1].
**Pitfall**: At the epipole (where view rays and light rays are parallel), the rectification mapping is singular and the derivative $ds/d\gamma$ goes to infinity. This causes large values that disrupt the Singular Value Decomposition (SVD) [S1].
**Complexity**: Rectification of the camera depth map and unrectification take $O(sr)$ time, and rectification of the shadow map takes $O(sd)$ time, where $s$ is the number of slices, $r$ is the average number of camera rays per slice, and $d$ is the number of light rays [S1].

### Gaussian Transformation to World Space
**Definition**: Transforms Gaussian parameters predicted in camera space into world space for rendering, using a 4x4 camera-to-world extrinsics matrix $\mathbf{T}$ containing rotation $\mathbf{R}$ and translation $\mathbf{t}$ [S2].
**Formula**: The world-space mean is transformed as $\mu_{\text{world}} = \mathbf{T}\mu_{\text{cam}}$, applying the full 4x4 transformation matrix [S2]. The world-space covariance is transformed as $\Sigma_{\text{world}} = \mathbf{R}\Sigma_{\text{cam}}\mathbf{R}^T$, using only the 3x3 rotation component $\mathbf{R}$ [S2]. The world-space opacity is unchanged: $\alpha_{\text{world}} = \alpha_{\text{cam}}$ [S2]. The world-space scale is transformed as $\mathbf{s}_{\text{world}} = \mathbf{D}_R\mathbf{R}\mathbf{s}_{\text{cam}}$, where $\mathbf{D}_R$ is a block-diagonal matrix of Wigner D matrices that rotates spherical harmonics coefficients [S2].
**Parameter**: The library e3nn is used to compute the Wigner D matrices ($\mathbf{D}_R$) and to recompile the original 3D Gaussian splatting code base to follow e3nn's conventions [S2].

### Camera Rectification
**Definition**: A strategy to rotate visible camera view directions so they look at the center of a sub-region S, and align their distances to the center of S with a specific radius r, to ensure points inside S can be warped to meaningful coordinates [S3].
**Parameter**: r: The aligned distance to the region center, empirically set as the mean distance to the region center among the 1/4 nearest visible cameras [S3].
**Pitfall**: Visible cameras are not suitable for direct perspective warping because some do not fully cover the region or look at it obliquely, leading to meaningless warp coordinates [S3].

## Background
The construction of camera matrices from eye, center, and up vectors is a standard technique in computer graphics, often referred to as the "LookAt" function [G]. The formulation of NDC ray transformation for forward-facing scenes utilizing disparity sampling is detailed in [S1]. The transformation of Gaussian parameters for 3D Gaussian Splatting into world space, including Wigner D matrices for spherical harmonics, is described in [S2]. Camera rectification strategies for perspective warping are discussed in [S3].

## Sources
[S1] A hierarchical volumetric shadow algorithm for single scattering, https://doi.org/10.1145/1866158.1866200, fetched 2026-11-03
[S2] PixelSplat: 3D Gaussian Splats from Image Pairs for Scalable Generalizable 3D Reconstruction, https://doi.org/10.1109/cvpr52733.2024.01840, fetched 2026-10-26
[S3] F2-NeRF: Fast Neural Radiance Field Training with Free Camera Trajectories, https://doi.org/10.1109/cvpr52729.2023.00404, fetched 2026-10-26
