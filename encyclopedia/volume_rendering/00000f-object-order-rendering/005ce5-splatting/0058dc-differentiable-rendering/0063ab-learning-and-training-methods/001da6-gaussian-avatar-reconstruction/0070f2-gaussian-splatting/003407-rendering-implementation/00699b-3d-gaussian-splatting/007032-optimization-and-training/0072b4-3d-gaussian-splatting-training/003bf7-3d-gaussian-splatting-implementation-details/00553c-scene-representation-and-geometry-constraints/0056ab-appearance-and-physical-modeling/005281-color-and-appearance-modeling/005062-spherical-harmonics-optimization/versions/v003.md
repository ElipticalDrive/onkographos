# Spherical Harmonics Optimization

Also called: SH Optimization, SH Compression, SH Distillation, SH Quantization

## Overview
Spherical Harmonics (SH) Optimization in 3D Gaussian Splatting addresses the high memory bandwidth and computational costs associated with storing and evaluating high-degree SH coefficients for view-dependent color. It reduces parameter count, accelerates training and rendering, and enables efficient gradient-based camera tracking by optimizing the representation and evaluation of the radiance field. It sets itself apart from siblings like [Sorting and Rendering Pipeline Optimization](12180-sorting-and-rendering-pipeline-optimization.md) by targeting the data representation and gradient computation specific to the SH basis, rather than the rasterization pipeline itself.

## Shared principles
The 3D Gaussian Splatting representation stores view-dependent color as SH coefficients $\mathbf{c}_i$ for each Gaussian $i$. For degree $d$, there are $(d+1)^2$ coefficients per color channel. The standard implementation uses degree 3, resulting in 16 coefficients per channel (48 total for RGB, or 45 if the 0th-order base color is separated). The radiance $L(\mathbf{v})$ in direction $\mathbf{v}$ is computed by evaluating the SH basis functions $\mathbf{Y}(\mathbf{v})$ and taking the dot product with the coefficients:
$$ L(\mathbf{v}) = \sum_{c \in \{R,G,B\}} \sum_{j=0}^{(d+1)^2-1} c_{c,j} Y_j(\mathbf{v}) $$
Optimization methods modify this representation to reduce the dimensionality of $\mathbf{c}_i$, compress the storage of $\mathbf{c}_i$, or accelerate the evaluation of $L(\mathbf{v})$ and its gradients.

## Subtopics

### [First-order Spherical Harmonics Distillation](9467-first-order-spherical-harmonics-distillation.md)
First-order Spherical Harmonics Distillation reduces the view-dependent color representation from third-order to first-order SH to decrease parameter count and computation cost. It achieves this by distilling the radiance and depth information from a pretrained third-order teacher model into a compact first-order student model. Unlike standard degree reduction which may lose high-frequency detail, this method uses knowledge distillation to preserve rendering fidelity despite the aggressive compression. It sets itself apart from siblings like [Spherical Harmonics Vector Quantization](14181-spherical-harmonics-vector-quantization.md) by focusing on algebraic degree reduction via distillation rather than codebook-based quantization, and from [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md) by targeting the SH coefficient storage and evaluation cost directly. See [First-order Spherical Harmonics Distillation](9467-first-order-spherical-harmonics-distillation.md).

### [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md)
This method provides analytical derivatives of the 3D Gaussian Splatting (3DGS) rendering pipeline with respect to the camera pose $T_{CW} \in SE(3)$, enabling efficient gradient-based camera tracking in Simultaneous Localization and Mapping (SLAM) systems. Unlike automatic differentiation, which incurs significant overhead in performance-critical rasterization loops, this approach computes minimal Jacobians on the Lie group manifold, ensuring the derivative dimensionality matches the six degrees of freedom of the camera. It distinguishes itself from prior ray-intersection-based pose optimization methods by being applicable to the dense, EWA-splatting-based rasterization used in standard 3DGS. See [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md).

### [Efficient Multi-View Training for 3D Gaussian Splatting](13982-efficient-multi-view-training-for-3d-gaussian-splatting.md)
This method enables stable optimization of 3D Gaussian Splatting (3DGS) by processing multiple training images per iteration, addressing the high gradient variance inherent in the standard single-view approach. Unlike standard 3DGS which renders one image per step due to rasterization overhead, this method uses thread-efficient partial rendering to gather pixels from multiple views without idle thread waste, while replacing the incompatible Adaptive Density Control (ADC) with a 3D-space compatible densification strategy. See [Efficient Multi-View Training for 3D Gaussian Splatting](13982-efficient-multi-view-training-for-3d-gaussian-splatting.md).

### [Spherical Harmonics Vector Quantization](14181-spherical-harmonics-vector-quantization.md)
Spherical Harmonics Vector Quantization (SH VQ) is a compression technique for 3D Gaussian Splatting that replaces per-splat SH coefficient vectors with indices pointing to a shared palette of distinct SH vectors, significantly reducing memory footprint at the cost of minor lighting artifacts. It sets itself apart from other optimization methods by addressing the storage bottleneck of high-dimensional view-dependent data directly through clustering, rather than altering the rendering pipeline or gradient computation. See [Spherical Harmonics Vector Quantization](14181-spherical-harmonics-vector-quantization.md).

#### SH Degree Zero Optimization
- **Definition**: Setting the maximum degree of Spherical Harmonics (SH) coefficients to 0 to remove superfluous operations and improve training speed, as higher-degree SH coefficients for color remain unchanged during training [S1, unofficial].
- **Step**: Modify the model code to delete operations relative to the max degree in the model and set the max degree parameter to 0 [S1, unofficial].
- **Pitfall**: In the original 3D Gaussian Splatting model, features represented by SH coefficients with degree > 0 are unchanged during all training, making them useless [S1, unofficial].
- **Number**: Directly changing the max degree to 0 yields a 16.25% improvement compared to the default. With specific changing code, it yields a 16.57% improvement but increases CPU-GPU process time [S1, unofficial].

## Background
The standard 3D Gaussian Splatting method [12] uses degree 3 SH for view-dependent color, which is computationally expensive. Subsequent works identified that high-degree SH coefficients often do not change significantly during training or can be compressed without visible quality loss. The optimization of SH representations has become a key area for improving the efficiency of 3DGS in terms of memory, training speed, and rendering performance.

## Sources
[S1] I3DGS: Improve 3D Gaussian Splatting from Multiple Dimensions, https://arxiv.org/abs/2405.06408v1, fetched 2026-10-26 (unofficial)
