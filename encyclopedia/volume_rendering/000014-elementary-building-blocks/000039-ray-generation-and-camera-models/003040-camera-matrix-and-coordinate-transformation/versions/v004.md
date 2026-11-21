# Camera Matrix and Coordinate Transformation

Also called: Camera Pose, View Transformation, Ray Space Mapping

## Overview
This family defines the geometric mapping between world coordinates, camera-local coordinates, and normalized device coordinates (NDC) to enable accurate ray generation for volume rendering. It solves the problem of establishing a consistent coordinate frame for rays, ensuring that ray origins and directions are correctly oriented relative to the scene and the image plane. What sets this family apart from sibling methods like [Sky Gradient](1262-sky-gradient.md) or [Arcball Camera](1639-arcball-camera.md) is its focus on the fundamental linear algebra and coordinate system conversions required before any specific sampling strategy (e.g., depth, disparity, or angular) can be applied. It provides the mathematical foundation for [Camera Ray Generation](7456-camera-ray-generation.md).

## Shared principles
The core mechanism of this family is the application of affine transformations to map points and vectors between different reference frames. All methods in this family rely on the definition of a camera pose, typically represented by a $4 \times 4$ matrix or a set of basis vectors (eye, center, up).

1.  **Coordinate Frames**:
    *   **World Space**: The global coordinate system in which the scene geometry is defined.
    *   **Camera (View) Space**: A local coordinate system attached to the camera, typically with the origin at the eye position, the Z-axis aligned with the viewing direction (often $-Z$ or $+Z$), and the X/Y axes aligned with the image plane.
    *   **Normalized Device Coordinates (NDC)**: A canonical cube, typically $[-1, 1]^3$, used to standardize ray directions and origins for uniform sampling, particularly in forward-facing scenes.

2.  **Transformation Matrices**:
    *   **World-to-View ($M_{w2v}$)**: Transforms world coordinates to camera space. It is constructed from the camera's basis vectors and position.
    *   **Camera-to-World ($M_{c2w}$)**: The inverse of the world-to-view matrix. It transforms camera-space rays (origins and directions) into world space for intersection tests with scene geometry.
    *   **Projection/NDC Mapping**: Transforms camera-space coordinates into NDC space, often involving perspective division or disparity-based reparameterization.

3.  **Ray Representation**:
    *   Rays are defined by an origin $\mathbf{o}$ and a direction $\mathbf{d}$.
    *   When transforming rays, origins are treated as points (using homogeneous coordinate $w=1$), while directions are treated as vectors (using $w=0$) to ensure translation does not affect direction.

4.  **Basis Construction**:
    *   The camera basis vectors (Right/Up/Forward) are derived from the eye position, target point, and an up vector using cross products to ensure orthogonality.

## Subtopics

### [NDC Ray Transformation](905-ndc-ray-transformation.md)
Maps camera-space rays into a normalized device coordinate (NDC) cube $[-1, 1]^3$ to enable uniform sampling in disparity (inverse depth) for forward-facing scenes with unbounded depth. Unlike standard ray generation which samples metric depth linearly, this method shifts ray origins to the near plane and projects them such that linear sampling in the transformed NDC parameter $t'$ corresponds to linear sampling in disparity from the near plane to infinity. This allows rendering scenes extending to infinity within a fixed bounding volume, a requirement for real-world forward-facing captures like those in the LLFF dataset. Builds on the geometric definition of rays and the parallel execution model from [Ray generation and camera models](770-ray-generation-and-camera-models.md).

### [View Matrix Construction](1644-view-matrix-construction.md)
Builds a $4 \times 4$ transformation matrix that maps coordinates from world space to camera (view) space, or conversely transforms rays from view space to world space, given an eye position, a target center, and an up vector. It solves the problem of defining a canonical camera coordinate frame for ray generation and volume rendering [G]. Unlike [Camera Pose Transformation](1672-camera-pose-transformation.md), which applies a rigid body transform to an existing pose, this method constructs the basis from scratch. It differs from [Orbit Camera Basis Computation](1749-orbit-camera-basis-computation.md) by accepting arbitrary eye/target/up inputs rather than deriving them from spherical angles.

### [Point to Camera Transformation](1660-point-to-camera-transformation.md)
The Point to Camera Transformation converts 3D points from world coordinates into the camera's local coordinate system. It establishes the relative position of scene geometry with respect to the viewer, serving as a foundational step for ray generation and visibility checks [G]. Unlike [NDC Ray Transformation](905-ndc-ray-transformation.md), which maps pixels to rays, or [View Matrix Construction](1644-view-matrix-construction.md), which builds the full affine transform matrix, this method explicitly computes the rotated and translated position of individual points to facilitate downstream operations like depth sampling or projection [G].

### Epipolar Geometry Rectification
*   **Definition**: A coordinate transformation that maps 3D points to $(\alpha, \beta, \gamma)$ where $\alpha$ specifies the epipolar slice, $\beta$ is the angle of the view ray relative to the light direction, and $\gamma$ is the distance from the light ray through the eye point. This allows view rays and light rays to be indexed independently along orthogonal axes [S1].
*   **Step**: To handle the epipole singularity, render results within a small ring (about two degrees) around the epipole using a standard ray marcher, and apply the epipolar algorithm only to the remainder of the image [S1].
*   **Pitfall**: At the epipole (where view rays and light rays are parallel), the rectification mapping is singular and the derivative $ds/d\gamma$ goes to infinity. This causes large values that disrupt the Singular Value Decomposition (SVD) [S1].
*   **Complexity**: Rectification of the camera depth map and unrectification take $O(sr)$ time, and rectification of the shadow map takes $O(sd)$ time, where $s$ is the number of slices, $r$ is the average number of camera rays per slice, and $d$ is the number of light rays [S1].

### Gaussian Transformation to World Space
*   **Definition**: Transforms Gaussian parameters predicted in camera space into world space for rendering, using a $4 \times 4$ camera-to-world extrinsics matrix $\mathbf{T}$ containing rotation $\mathbf{R}$ and translation $\mathbf{t}$ [S2].
*   **Formula**: The world-space mean is transformed as $\mu_{\text{world}} = \mathbf{T}\mu_{\text{cam}}$, applying the full $4 \times 4$ transformation matrix [S2]. The world-space covariance is transformed as $\Sigma_{\text{world}} = \mathbf{R}\Sigma_{\text{cam}}\mathbf{R}^T$, using only the $3 \times 3$ rotation component $\mathbf{R}$ [S2]. The world-space opacity is unchanged: $\alpha_{\text{world}} = \alpha_{\text{cam}}$ [S2]. The world-space scale is transformed as $\mathbf{s}_{\text{world}} = \mathbf{D}\mathbf{R}\mathbf{s}_{\text{cam}}$, where $\mathbf{D}$ is a block-diagonal matrix of Wigner D matrices that rotates spherical harmonics coefficients [S2].
*   **Parameter**: The library `e3nn` is used to compute the Wigner D matrices ($\mathbf{D}$) and to recompile the original 3D Gaussian splatting code base to follow `e3nn`'s conventions [S2].

### Camera Rectification
*   **Definition**: A strategy to rotate visible camera view directions so they look at the center of a sub-region $S$, and align their distances to the center of $S$ with a specific radius $r$, to ensure points inside $S$ can be warped to meaningful coordinates [S3].
*   **Parameter**: $r$: The aligned distance to the region center, empirically set as the mean distance to the region center among the 1/4 nearest visible cameras [S3].
*   **Pitfall**: Visible cameras are not suitable for direct perspective warping because some do not fully cover the region or look at it obliquely, leading to meaningless warp coordinates [S3].

### Camera Parameter Conversion to Open3D
*   **Definition**: Converts a stack of viewpoint cameras (with projection matrices and world view transforms) into Open3D `PinholeCameraParameters` objects [S4, unofficial].
*   **Step**: For each viewpoint, construct an NDC-to-pixel transformation matrix `ndc2pix` of shape $(3,4)$: `[[W/2, 0, 0, (W-1)/2], [0, H/2, 0, (H-1)/2], [0, 0, 0, 1]]` [S4, unofficial]. Compute the intrinsic matrix by multiplying the projection matrix by `ndc2pix` and transposing the result, then taking the top-left $3 \times 3$: `intrins = (viewpoint_cam.projection_matrix @ ndc2pix)[:3,:3].T` [S4, unofficial]. Extract intrinsic parameters: `cx = intrins[0,2]`, `cy = intrins[1,2]`, `fx = intrins[0,0]`, `fy = intrins[1,1]` [S4, unofficial]. Create an Open3D `PinholeCameraIntrinsic` object with width, height, and the extracted $f_x, f_y, c_x, c_y$ [S4, unofficial]. Compute the extrinsic matrix as the transpose of the world view transform: `extrinsic = np.asarray((viewpoint_cam.world_view_transform.T).cpu().numpy())` [S4, unofficial]. Create an Open3D `PinholeCameraParameters` object, set its intrinsic and extrinsic attributes, and append to the trajectory list [S4, unofficial].

## Background
The concept of the view matrix and coordinate transformation originates from classical computer graphics and photogrammetry. The construction of a camera basis from eye, center, and up vectors is a standard technique described in early graphics literature, such as Foley et al.'s "Computer Graphics: Principles and Practice" [S5, unofficial]. The specific application of NDC transformations for disparity sampling in neural radiance fields (NeRF) was popularized by the LLFF (Local Light Field Fusion) paper [S1], which introduced the NDC ray transformation to handle forward-facing scenes with unbounded depth. The transformation of Gaussian parameters for 3D Gaussian Splatting, as detailed in PixelSplat [S2], extends these linear algebra concepts to handle the covariance and spherical harmonics of Gaussian primitives. Camera rectification strategies for multi-view stereo and NeRF training, as seen in F2-NeRF [S3], further refine these transformations to optimize sampling efficiency in specific sub-regions.

## Sources
[S1] A hierarchical volumetric shadow algorithm for single scattering, https://doi.org/10.1145/1866158.1866200, fetched 2026-11-03
[S2] PixelSplat: 3D Gaussian Splats from Image Pairs for Scalable Generalizable 3D Reconstruction, https://doi.org/10.1109/cvpr52733.2024.01840, fetched 2026-10-26
[S3] F2-NeRF: Fast Neural Radiance Field Training with Free Camera Trajectories, https://doi.org/10.1109/cvpr52729.2023.00404, fetched 2026-10-26
[S4] Zhaoyibinn/VGVGGT, https://github.com/Zhaoyibinn/VGVGGT, fetched 2026-10-27 (unofficial)
[S5] Computer Graphics: Principles and Practice, Foley et al. (unofficial)
