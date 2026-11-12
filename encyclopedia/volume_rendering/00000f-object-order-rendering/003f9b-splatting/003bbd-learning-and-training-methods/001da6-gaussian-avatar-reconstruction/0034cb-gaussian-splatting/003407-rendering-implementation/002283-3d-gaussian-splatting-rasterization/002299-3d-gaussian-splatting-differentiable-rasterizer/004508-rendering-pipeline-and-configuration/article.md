# Rendering Pipeline and Configuration

Also called: Gaussian Rasterization Configuration, 3DGS Pipeline Setup

## Overview
This family defines the configuration structures and input validation logic required to execute the differentiable rasterization of 3D Gaussian Splatting. It solves the problem of managing mutually exclusive representations (e.g., Spherical Harmonics vs. pre-computed RGB, parametric scales vs. pre-computed covariances) and encapsulating camera parameters into a single context for the CUDA backend. It sets itself apart from the core rasterization algorithm by focusing on data integrity and parameter bundling rather than the geometric projection or compositing steps themselves.

## Shared principles
All components in this family adhere to the following mechanisms:
1.  **Mutually Exclusive Representations**: The system enforces that exactly one representation is active for specific Gaussian attributes. Specifically, either Spherical Harmonics (`shs`) or pre-computed RGB (`colors_precomp`) must be provided, but not both. Similarly, either parametric scales/rotations (`scales`, `rotations`) or pre-computed 3D covariances (`cov3D_precomp`) must be provided, but not both [S1, S2, unofficial].
2.  **Context Encapsulation**: Camera intrinsics, extrinsics, and rasterization flags are bundled into a single configuration object (`GaussianRasterizationSettings`) to avoid passing disparate parameters individually to the CUDA kernel [Parent].
3.  **Empty Tensor Handling**: Missing inputs are not treated as errors if a valid alternative representation exists; instead, they are replaced with empty tensors of appropriate shape (e.g., `(N, 0)`) to maintain tensor consistency for the CUDA backend [S1, S2, unofficial].
4.  **Device Consistency**: All configuration tensors (background color, view/projection matrices) must reside on the same device (typically CUDA) as the Gaussian data to prevent kernel launch errors [S1, S4, unofficial].

## Subtopics

### [GaussianRasterizer Initialization and Input Validation](2923-gaussianrasterizer-initialization-and-input-validation.md)
Validates input tensor integrity, ensuring mutually exclusive representations are correctly set and missing inputs are normalized to empty tensors. Choose this when implementing the pre-rasterization data preparation step to prevent runtime errors.

### [GaussianRasterizationSettings](2924-gaussianrasterizationsettings.md)
A configuration structure encapsulating camera intrinsics, extrinsics, and rasterization flags. Choose this when defining the context object passed to the rasterization kernel.

### [Gaussian Splatting Rendering Pipeline](3415-gaussian-splatting-rendering-pipeline.md)
Orchestrates the full execution flow, including covariance reconstruction, 2D projection, color evaluation, and alpha compositing. Choose this when implementing the complete rendering loop for a single camera view.

## Background
The configuration and validation logic originates from the original 3D Gaussian Splatting implementation by Kerbl et al. [2023] [S1, S6, unofficial]. The pipeline was designed to support interactive frame rates by offloading heavy computation to CUDA while maintaining a Python-side interface for data preparation and gradient management. The use of `GaussianRasterizationSettings` and strict input validation was introduced to handle the complexity of differentiable rendering with multiple representation options (SH vs. RGB, parametric vs. pre-computed covariance) [Parent].

## Sources
[S1] 3D Gaussian Splatting GitHub Repository, https://github.com/graphdeco-inria/gaussian-splatting, 2023-10-10
[S2] 3D Gaussian Splatting Implementation Notes, https://github.com/graphdeco-inria/gaussian-splatting/blob/main/utils/sh_utils.py, 2023-10-10
[S3] 3D Gaussian Splatting Technical Report, https://arxiv.org/abs/2308.04079, 2023-08-07
[S4] 3D Gaussian Splatting Pipeline Implementation, https://github.com/graphdeco-inria/gaussian-splatting/blob/main/utils/camera_utils.py, 2023-10-10
[S5] 3D Gaussian Splatting Rasterization Details, https://github.com/graphdeco-inria/gaussian-splatting/blob/main/rasterize_points.cu, 2023-10-10
[S6] 3D Gaussian Splatting Official Documentation, https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/, 2023-10-10
[S7] 3D Gaussian Splatting Depth Rendering Extension, https://github.com/graphdeco-inria/gaussian-splatting/blob/main/utils/loss_utils.py, 2023-10-10
