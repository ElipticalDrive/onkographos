# Neural Light Field Representations


## Overview
Neural Light Field Representations are a family of scene representation methods that model appearance and lighting to enable novel view synthesis and intrinsic decomposition. [G] They differ from standard NeRF by explicitly decoupling view-independent geometry/material properties from view-dependent lighting effects, often using direct ray-to-color mapping or progressive network connectivity to improve training stability and rendering speed. [G]

## Shared principles
All methods in this family build on [Specific Model Architectures](8585-specific-model-architectures.md). [G] While the specific decomposition varies, the core principle is the decoupling of view-independent geometric/material features from view-dependent lighting effects. [G]

**Input Representation:**
Raw coordinates $\mathbf{x}$ and viewing/lighting directions $\mathbf{d}$ are mapped to a higher-dimensional space using positional encoding $\gamma(\cdot)$ to mitigate spectral bias. [G]
$$
\gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
$$
where $L$ is the number of frequency bands. [G]

**Network Topology:**
The canonical implementation uses coupled Multi-Layer Perceptrons (MLPs) modified to output intermediate physical properties rather than raw color. [G]
1. **Geometry/Structure Network:** Takes $\gamma(\mathbf{x})$ as input. It typically consists of hidden layers with ReLU activations and skip connections. [G] It outputs signed distance field (SDF) values, normals, shading, and reflectance. [S1]
2. **Appearance/Lighting Network:** Takes the latent features, $\gamma(\mathbf{x})$, and $\mathbf{d}$ (or derived directions) as input. It computes the final color $\mathbf{c}$ by combining material properties with lighting models (e.g., light position, shading functions). [G]

**Rendering Integration:**
During training, rays are sampled at points $\mathbf{x}_i$. [G] The MLPs predict $\sigma_i$ and material properties. [G] The pixel color $C(\mathbf{r})$ is computed via volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
where $\delta_i$ is the distance between samples. [G] The color $\mathbf{c}_i$ is derived from the decomposed components specific to the child method. [G]

## Subtopics

### [Neural Light Field (NeLF)](9257-neural-light-field-nelf.md)
Neural Light Field (NeLF) is a scene representation method that maps camera rays directly to RGB color values, bypassing the volumetric ray-marching and alpha-composition steps required by Neural Radiance Fields (NeRF). [G] Unlike NeRF, which requires multiple forward passes per ray to aggregate radiance, NeLF achieves novel view synthesis with a single forward pass per ray, enabling rendering speeds that are orders of magnitude faster. [G] While standard NeLF implementations often use shallow networks, efficient variants like R2L employ deep residual MLPs trained via knowledge distillation from a pre-trained NeRF teacher to maintain high visual fidelity. [G] The method builds on [Specific Model Architectures](8585-specific-model-architectures.md) by decoupling view-independent geometric features from view-dependent lighting effects. [G]

### [Progressively-connected Light Field Network (ProLiF) Architecture](17211-progressively-connected-light-field-network-prolif-architect.md)
ProLiF is a neural light field representation that decouples view-independent geometry and material properties from view-dependent lighting, enabling efficient view synthesis by training with a progressively connected architecture. [G] It sets itself apart from standard NeRF or NeLF by replacing a single monolithic MLP with a set of independent subnetworks that are merged during training, enforcing an inductive bias for consistent scene geometry at early stages to prevent overfitting and improve detail reconstruction. [G]

### Neural Multi-Light Implicit Representation (Stage 1)
An extension of Neuralangelo that incorporates light position as an input to the network to enable multi-light implicit representation. [S1] Input includes camera pose, light position, and points $(x, y, z)$. [S1] Training time for Stage 1 follows Neuralangelo training time, while Stage 2 takes 19 hours. [S1] Experiments were conducted on a single Nvidia RTX 3090. [S1]

### LANe
Trains on a single scene under varying lighting conditions to learn adaptation by creating a light field. [S2] Uses a corresponding shader to modulate object appearances for coherent integration into scenes with different lighting. [S2]

### Neural Light Transport (NLT)
Method using neural networks to learn light transport in static and dynamic 3D scenes, supporting inference in both 3D and 2D spaces for global illumination and 3D scene geometry processing. [S3, unofficial] Architecture is U-Net based with downsampling stride 2 via convolutions, upsampling via linear interpolation, and 3 skip connections. [S3, unofficial] Skip connection and concatenation layers store input images and two intermediate feature maps, increasing memory access overhead. [S3, unofficial] Compared to existing 2D image-domain methods, NLT supports inference in both 3D and 2D spaces, achieving global illumination effects and 3D scene geometry processing. [S3, unofficial]

## Background
Neural Light Field representations emerged as a response to the computational bottleneck of volumetric rendering in NeRF. [G] The foundational NeLF paper introduced the direct mapping of rays to colors. [G] Subsequent work like ProLiF addressed the stability and capacity limits of direct mapping by introducing progressive connectivity. [G] More recent variants like MLI-NeRF extend these representations to handle complex lighting conditions by integrating light position into the implicit representation. [S1]

## Sources
[S1] MLI-NeRF: Multi-Light Intrinsic-Aware Neural Radiance Fields, https://arxiv.org/abs/2411.17235v1, fetched 2026-10-26
[S2] Semantically-aware Neural Radiance Fields for Visual Scene Understanding: A Comprehensive Review, https://link.springer.com/article/10.1007/s11263-025-02663-5, fetched 2026-10-27
[S3] Neural Rendering and Its Hardware Acceleration: A Review, https://doi.org/10.48550/arxiv.2402.00028, fetched 2026-10-26 (unofficial)
