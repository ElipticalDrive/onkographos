# Neural Light Field Representations

Also called: Neural Light Fields, NeLF, ProLiF, MLI-NeRF

## Overview
Neural Light Field Representations map camera rays directly to RGB color values, bypassing the volumetric ray-marching and alpha-composition steps required by Neural Radiance Fields (NeRF) [S1]. Unlike NeRF, which requires multiple forward passes per ray to aggregate radiance, these methods achieve novel view synthesis with a single forward pass per ray, enabling rendering speeds that are orders of magnitude faster .

## Shared principles
All methods in this family build on [Specific Model Architectures](8585-specific-model-architectures.md). While the specific decomposition varies, the core principle is the decoupling of view-independent geometric/material features from view-dependent lighting effects.

**Input Representation:**
Raw coordinates $\mathbf{x}$ and $\mathbf{d}$ are mapped to a higher-dimensional space using positional encoding $\gamma(\cdot)$ to mitigate spectral bias [S1].
$$
\gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
$$
where $L$ is the number of frequency bands [S1].

**Network Topology:**
The canonical implementation uses coupled Multi-Layer Perceptrons (MLPs). The density network $F_\sigma$ and color network $F_c$ are modified to output intermediate physical properties rather than raw color [S1].
1.  **Density/Geometry Network:** Takes $\gamma(\mathbf{x})$ as input. It consists of $L$ hidden layers with ReLU activations and skip connections from the input to every 4th layer [S1]. It outputs density $\sigma$ and latent features representing material properties (e.g., normals, roughness, albedo) [S1].
2.  **Appearance/Lighting Network:** Takes the latent features, $\gamma(\mathbf{x})$, and $\mathbf{d}$ (or derived directions like reflection vectors) as input. It computes the final color $\mathbf{c}$ by combining material properties with lighting models (e.g., environment maps, specular models) [S1].

**Rendering Integration:**
During training, rays are sampled at points $\mathbf{x}_i$. The MLPs predict $\sigma_i$ and material properties. The pixel color $C(\mathbf{r})$ is computed via volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
where $\delta_i$ is the distance between samples [S1]. The color $\mathbf{c}_i$ is derived from the decomposed components specific to the child method.

## Subtopics

### [Neural Light Field (NeLF)](9257-neural-light-field-nelf.md)
Neural Light Field (NeLF) is a scene representation method that maps camera rays directly to RGB color values, bypassing the volumetric ray-marching and alpha-composition steps required by Neural Radiance Fields (NeRF). Unlike NeRF, which requires multiple forward passes per ray to aggregate radiance, NeLF achieves novel view synthesis with a single forward pass per ray, enabling rendering speeds that are orders of magnitude faster. While standard NeLF implementations often use shallow networks, efficient variants like R2L employ deep residual MLPs trained via knowledge distillation from a pre-trained NeRF teacher to maintain high visual fidelity.

### [Progressively-connected Light Field Network (ProLiF) Architecture](17211-progressively-connected-light-field-network-prolif-architect.md)
ProLiF is a neural light field representation that decouples geometry and appearance via coupled MLPs, optimized through a progressive training scheme that incrementally increases network connectivity. It differs from sibling methods like NeLF by training with a fixed total parameter budget that is redistributed from many small subnetworks to fewer, wider networks over time, preventing overfitting in early stages while enabling high-fidelity detail reconstruction in later stages.

### Neural Multi-Light Implicit Representation (Stage 1)
An extension of Neuralangelo that incorporates light position as an input to the network to enable multi-light implicit representation [S1]. Input includes camera pose, light position, and points $(x, y, z)$ [S1]. Training time for Stage 1 follows Neuralangelo training time, while Stage 2 takes 19 hours [S1]. Experiments were conducted on a single Nvidia RTX 3090 [S1].

## Background
Neural Light Field representations emerged as a response to the computational bottleneck of volumetric rendering in NeRF . The foundational NeLF paper introduced the direct mapping of rays to colors . Subsequent work like ProLiF addressed the stability and capacity limits of direct mapping by introducing progressive connectivity [S1]. More recent variants like MLI-NeRF extend these representations to handle complex lighting conditions by integrating light position into the implicit representation [S1].

## Sources
[S1] MLI-NeRF: Multi-Light Intrinsic-Aware Neural Radiance Fields, https://arxiv.org/abs/2411.17235v1, fetched 2026-10-26
[S2] Neural Light Fields, https://arxiv.org/abs/2003.08933, fetched 2026-10-26
[S3] Residual Neural Light Fields, https://arxiv.org/abs/2104.09125, fetched 2026-10-26
