# Multi-scale filtering

Also called: Multi-scale Gaussian filtering, MIP-Gaussian filtering

## Overview
Multi-scale filtering addresses spatial aliasing and performance degradation in 3D Gaussian Splatting (3DGS) when rendering at varying resolutions or camera distances. Unlike standard 3DGS, which suffers from dilation/erosion artifacts and high-frequency noise due to fixed Gaussian scales, multi-scale filtering constrains the representation's frequency content. It achieves this through 3D smoothing during optimization and resolution-aware selection or filtering during rendering, ensuring alias-free images across sampling rates [S1].

## Shared principles
All methods in this family share the goal of matching the Gaussian representation's frequency to the sampling rate (resolution) of the rendered image. The core problem is that standard 3DGS Gaussians are optimized for a specific training resolution; when rendered at different scales, they either alias (if too small/high-frequency) or blur excessively (if too large/low-frequency).

The shared mathematical foundation involves the 2D Gaussian projection used in splatting. A 2D Gaussian $G_{2D}$ with covariance $\Sigma_{2D}$ is typically rendered as:
$$ G_{2D}(x) = e^{-\frac{1}{2} x^T \Sigma_{2D}^{-1} x} $$
Multi-scale methods modify this either by:
1.  **Pre-filtering:** Increasing the variance of $\Sigma_{2D}$ (or the underlying 3D $\Sigma$) to act as a low-pass filter, approximating the camera's pixel box filter [S1].
2.  **Selection:** Dynamically selecting Gaussians based on their projected pixel coverage relative to the current resolution, discarding those that are too small (aliasing) or too large (unnecessary computation) [S1].

## Subtopics

### [MIP-Splatting](6063-mip-splatting.md)
MIP-Splatting eliminates spatial aliasing by introducing a 3D smoothing filter during optimization and a 2D Mip filter during rendering [6063-mip-splatting.md]. It calculates the maximal sampling frequency $\hat{\nu}$ based on the training views and applies a Gaussian low-pass filter to the 3D covariance $\Sigma_k$ to constrain frequency to the Nyquist limit. During rendering, it replaces standard dilation with a 2D Mip filter that adds variance $s$ to the projected covariance $\Sigma_{k2D}$ to approximate the physical pixel footprint.

### [Multi-scale 3D Gaussian Splatting](8083-multi-scale-3d-gaussian-splatting.md)
Multi-scale 3D Gaussian Splatting (MS 3DGS) mitigates aliasing and performance issues by maintaining a multi-scale representation [8083-multi-scale-3d-gaussian-splatting.md]. It aggregates small, high-frequency Gaussians into larger, low-frequency ones during training using voxel-based average pooling. During rendering, it selectively renders Gaussians based on their pixel coverage relative to stored bounds ($S_k^{max}, S_k^{min}$) and the current resolution scale, filtering out those that are too small (aliasing) or too large (blur).

### Progressive Low-Pass Filtering for 3DGS
Progressively applies low-pass filtering to avoid 2D Gaussian projections becoming smaller than a pixel [S1].

## Background
The concept of multi-scale filtering in 3DGS stems from the observation that standard 3DGS optimization does not account for the Nyquist limit relative to the rendering resolution, leading to aliasing artifacts when zooming or changing resolution [S1]. MIP-Splatting [6063-mip-splatting.md] was one of the first to explicitly model the camera's box filter via 2D Mip filtering. Multi-scale 3D Gaussian Splatting [8083-multi-scale-3d-gaussian-splatting.md] extended this by creating explicit hierarchical levels of detail. These methods are cited in surveys on recent advances in 3D Gaussian Splatting [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
