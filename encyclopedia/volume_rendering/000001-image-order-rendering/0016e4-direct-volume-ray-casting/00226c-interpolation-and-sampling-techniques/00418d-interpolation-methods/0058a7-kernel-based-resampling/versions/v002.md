# Kernel-based Resampling

Also called: Kernel Interpolation, Convolution-based Interpolation, Weighted Sampling

## Overview
Kernel-based resampling approximates a continuous volumetric field $V(\mathbf{p})$ from discrete samples $V_i$ by convolving the data with a spatially localized weight function (kernel) centered at the query position $\mathbf{p}$ along the viewing ray. It solves the discretization artifacts inherent in nearest-neighbor sampling by providing a continuous, differentiable estimate of the volume value. It sets itself apart from sibling methods like [Nearest Neighbor Sampling](11694-nearest-neighbor-sampling.md) by utilizing multiple neighbors to reduce aliasing, and from [Gradient-Based Interpolation by Sample Splatting](9072-gradient-based-interpolation-by-sample-splatting.md) by relying on scalar weight functions rather than geometric splatting primitives.

## Shared principles
All methods in this family operate on the principle that a continuous function $V(\mathbf{p})$ must be approximated from a discrete set of samples $V_i$ located at grid positions $\mathbf{p}_i$. The core mechanism involves:

1.  **Query Positioning**: Determining the 3D coordinate $\mathbf{p}$ along the ray $r(t) = \mathbf{o} + t\mathbf{d}$ where a value is needed. This may involve converting depth parameters to physical distances or warping coordinates for neural encodings.
2.  **Kernel/Weight Application**: Computing weights $w_i(\mathbf{p})$ based on the distance between $\mathbf{p}$ and the sample centers $\mathbf{p}_i$. The interpolated value is:
    $$
    V(\mathbf{p}) = \sum_{i} w_i(\mathbf{p}) V_i
    $$
    The choice of kernel (e.g., linear, cubic, inverse distance) determines the spatial extent (number of neighbors) and frequency response (aliasing vs. blurring) .
3.  **Coordinate Transformation**: Mapping world-space or camera-space coordinates to the local coordinate system of the volume data (e.g., normalized $[-1, 1]$ or $[0, 1]$ ranges) before interpolation .

## Subtopics

### [Interpolation Schemes for Volume Rendering](817-interpolation-schemes-for-volume-rendering.md)
Encompasses the full spectrum from zero-order box filters to high-order frequency-domain sinc resampling, including specialized hardware approximations and fast GPU-accelerated cubic evaluations. Chosen when a broad range of fidelity/performance trade-offs is needed.

### [Lanczos Resampling](11246-lanczos-resampling.md)
Uses a windowed sinc function to balance sharpness with ringing suppression. Chosen when higher fidelity than linear or cubic interpolation is required, particularly in medical imaging.

### [Interpolation Functions for Resampling](15932-interpolation-functions-for-resampling.md)
Focuses on the trade-off between grey-value reconstruction fidelity and edge location accuracy. Chosen when negative values or gradient errors produce visible artifacts in medical imaging contexts.

## Background
Kernel-based resampling is a fundamental technique in digital signal processing and computer graphics, rooted in the Whittaker–Shannon interpolation formula which states that a band-limited signal can be perfectly reconstructed from its samples using a sinc kernel . In volume rendering, exact sinc reconstruction is computationally prohibitive due to its infinite support, leading to the development of truncated and windowed kernels (such as Lanczos) and polynomial approximations (such as B-splines) . The method is widely used in direct volume ray casting to ensure smooth visual transitions and accurate gradient estimation for lighting calculations.

## Sources
[S1] Unofficial notes on Kernel-based Resampling, URL not provided, date fetched not provided.
[S2] Unofficial notes on Kernel-based Resampling, URL not provided, date fetched not provided.
