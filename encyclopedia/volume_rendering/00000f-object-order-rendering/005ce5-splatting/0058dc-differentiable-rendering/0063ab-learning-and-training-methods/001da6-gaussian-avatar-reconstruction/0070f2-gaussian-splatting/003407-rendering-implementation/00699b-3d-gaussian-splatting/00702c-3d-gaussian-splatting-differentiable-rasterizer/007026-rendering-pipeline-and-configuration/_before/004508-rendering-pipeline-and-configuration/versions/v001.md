# Rendering Pipeline and Configuration

Also called: GaussianRasterizer Setup, Rasterization Configuration

## Overview
The Rendering Pipeline and Configuration family manages the data preparation, camera context, and orchestration required to invoke the 3D Gaussian Splatting differentiable rasterizer. It solves the problem of translating learned 3D Gaussian primitives and camera parameters into a valid state for the CUDA kernel, ensuring mutual exclusivity of input representations and correct projection geometry. It differs from its siblings by providing the high-level orchestration and configuration structures rather than the low-level kernel logic or specific optimization variants.

## Shared principles
All components in this family operate on the principle of strict input validation and context encapsulation. The pipeline ensures that exactly one color representation (Spherical Harmonics or pre-computed RGB) and exactly one covariance representation (parametric scales/rotations or pre-computed 3D covariance) are active. The configuration structure encapsulates camera intrinsics, extrinsics, and rasterization flags, passing them as a single context object to the rasterizer to avoid redundant computation in the rendering loop.

## Subtopics

### [GaussianRasterizer Initialization and Input Validation](2923-gaussianrasterizer-initialization-and-input-validation.md)
Prepares input tensors by enforcing mutual exclusivity constraints and normalizing missing data into empty tensors to prevent undefined behavior during the forward pass.

### [GaussianRasterizationSettings](2924-gaussianrasterizationsettings.md)
A configuration data structure encapsulating camera parameters, background color, and rasterization flags, serving as the bridge between scene state and the CUDA kernel.

### [Gaussian Splatting Rendering Pipeline](3415-gaussian-splatting-rendering-pipeline.md)
Orchestrates the final stage of rendering, converting 3D Gaussians into a 2D image by handling covariance reconstruction, projection, SH evaluation, and invoking the rasterizer.

## Background
The rendering pipeline and configuration structures are integral to the implementation of 3D Gaussian Splatting [S1, S3, S4]. They were developed to support the differentiable rasterization backend, allowing gradients to flow back to Gaussian parameters during training [S2, S4]. The specific validation logic and settings structure are derived from the standard PyTorch implementation of the 3D Gaussian Splatting rasterizer [S1, S2, unofficial].

## Sources
[S1] 3D Gaussian Splatting GitHub Repository, https://github.com/graphdeco-inria/gaussian-splatting, 2023-10-10
[S2] 3D Gaussian Splatting Official Implementation, https://github.com/graphdeco-inria/gaussian-splatting/blob/main/utils/sh_utils.py, 2023-10-10
[S3] Kerbl, B., Kopanas, G., Leimkühler, T., & Drettakis, G. (2023). 3D Gaussian Splatting for Real-Time Radiance Field Rendering. ACM Transactions on Graphics, 42(4).
[S4] 3D Gaussian Splatting Technical Report, https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/, 2023-07-10
