# Rendering Pipeline and Configuration

Also called: GaussianRasterizer, GaussianRasterizationSettings, Gaussian Splatting Rendering Pipeline

## Overview
The Rendering Pipeline and Configuration family manages the high-level orchestration, input validation, and parameter bundling required to invoke the differentiable 3D Gaussian Splatting rasterizer. It solves the problem of heterogeneous input formats (e.g., precomputed covariances vs. parametric scales/rotations, SH vs. precomputed colors) by enforcing mutual exclusivity constraints and normalizing tensors before kernel launch. It sets itself apart from the raw CUDA kernel by handling PyTorch tensor management, error checking, and the construction of the argument tuple passed to the backend [S1, S2, S3, unofficial].

## Shared principles
The family enforces strict mutual exclusivity for scene representation to ensure deterministic kernel behavior:
1.  **Color Representation**: Exactly one of spherical harmonic coefficients (`shs`) or precomputed RGB colors (`colors_precomp`) must be provided. If both are empty/None or both are non-empty, an exception is raised [S1, S2, S3, unofficial].
2.  **Geometry Representation**: Exactly one of the parametric pair (`scales`, `rotations`) or precomputed 3D covariance matrices (`cov3D_precomp`) must be provided. If both are empty/None or both are non-empty, an exception is raised [S1, S2, S3, unofficial].
3.  **Device Consistency**: All input tensors must reside on the same device (typically CUDA) to prevent kernel launch errors [Parent].
4.  **Empty Tensor Normalization**: Any input tensor that is `None` is replaced with an empty `torch.Tensor([])` or a tensor of shape $(N, 0)$ to maintain batch consistency and prevent `None` propagation into the CUDA kernel [S1, S2, S3, unofficial].

## Subtopics

### [GaussianRasterizer Initialization and Input Validation](2923-gaussianrasterizer-initialization-and-input-validation.md)
The PyTorch wrapper that validates inputs, normalizes tensor formats, and invokes the backend. It differs from the raw CUDA kernel by handling high-level PyTorch tensor management, error checking, and the construction of the argument tuple passed to `_C.rasterize_gaussians`.

### [GaussianRasterizationSettings](2924-gaussianrasterizationsettings.md)
A configuration structure (NamedTuple/struct) bundling camera intrinsics, extrinsics, and rasterization flags. It serves as the primary input interface for the rendering pipeline, defining how 3D Gaussians are projected and composited onto the 2D image plane.

### [Gaussian Splatting Rendering Pipeline](3415-gaussian-splatting-rendering-pipeline.md)
The high-level orchestrator that executes covariance reconstruction, 2D projection, color evaluation, tile intersection, and alpha compositing. It sets itself apart from standard polygon rasterization by handling soft, overlapping volumetric primitives without geometry clipping or complex shader pipelines.

## Background
The pipeline architecture originates from the original 3D Gaussian Splatting implementation [S1]. It has been extended in variants such as 4D Gaussian Splatting [S2] and FastGS/Laplacian GS [S3] to support time-varying scenes and metric accumulation, respectively. It is used in all differentiable 3D Gaussian splatting applications requiring interactive frame rates and gradient-based optimization [S1].

## Sources
[S1] Kerbl, B., Kopanas, G., Leimkühler, T., & Drettakis, G. (2023). 3D Gaussian Splatting for Real-Time Radiance Field Rendering. ACM Transactions on Graphics (TOG).
[S2] 4D Gaussian Splatting Implementation Notes.
[S3] FastGS/Laplacian GS Implementation Notes.
[S4] Standard 3D GS Pipeline Implementation Notes.
[S5] Gaussian Splatting Rasterization Details.
[S6] 3D Gaussian Splatting Rasterization Details.
[S7] Gaussian Splatting Pipeline Configuration Notes.
