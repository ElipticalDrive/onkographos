# Geometric Priors and Depth Integration

Also called: Geometric Regularization in NeRF, Depth-Guided Radiance Fields

## Overview
This family encompasses methods that improve the geometric accuracy and structural consistency of Neural Radiance Fields (NeRFs), particularly under sparse-view or ill-posed conditions, by integrating explicit geometric constraints or external depth priors into the optimization loop. Unlike standard NeRF, which relies solely on photometric consistency and can suffer from geometric ambiguity, these methods enforce multi-view geometric consistency (e.g., epipolar constraints, feature matching) or leverage high-fidelity monocular depth estimates to anchor the 3D structure.

## Shared principles
The core mechanism remains the parametric function $F_\Theta$ that maps a 3D position $\mathbf{x} \in \mathbb{R}^3$ and a viewing direction $\mathbf{d} \in \mathbb{R}^3$ to a volume density $\sigma \in \mathbb{R}$ and an RGB color $\mathbf{c} \in \mathbb{R}^3$. This function is implemented as one or more neural networks, typically utilizing positional encoding $\gamma(\mathbf{x})$ to map raw coordinates to a higher-dimensional space using sinusoidal functions at increasing frequencies [Parent].

**Rendering Integration:**
During training, rays are sampled at points $\mathbf{x}_i$ along the ray. The networks predict $\sigma_i$ and $\mathbf{c}_i$. The pixel color $C(\mathbf{r})$ is computed via volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
and $\delta_i$ is the distance between samples [Parent].

**Input Representation:**
Raw coordinates are rarely used directly. They are mapped to a higher-dimensional space using positional encoding $\gamma(\mathbf{x})$. For a coordinate $\mathbf{x}$, the encoding includes the original value and sinusoidal functions at increasing frequencies:
$$
\gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
$$
where $L$ is the number of frequency bands [Parent].

## Subtopics

### [SfMNeRF](8664-sfmnerf.md)
Integrates Structure-from-Motion (SfM) constraints (epipolar geometry, feature matching) into the NeRF training loop to resolve geometric ambiguity in sparse views.

### [DASNeRF](8668-dasnerf.md)
Uses monocular depth priors (DepthAnythingV2) and dynamic sampling to enforce depth ordering and structural consistency, distinguishing itself from multi-view geometric methods by leveraging single-view depth fidelity.

### [GeoNeRF](#geonerf)

### GeoNeRF
- **Definition:** A model consisting of two parts: a geometric morphology reasoner and a renderer [S1]. The reasoner builds a cascaded cost volume for source view sequences. The renderer infers geometric and appearance representations using a Transformer-based attention mechanism and cascaded cost volumes, then renders fine images using volume rendering [S1].
- **Step:** Finally, volume rendering technology is used to render fine images [S1].

## Background
The integration of geometric priors addresses the inherent ambiguity in NeRF optimization, where photometric loss alone may lead to incorrect geometry, especially with few input images. Early work like SfMNeRF introduced multi-view geometric constraints to anchor 3D coordinates [S1]. Later approaches like DASNeRF shifted towards leveraging advanced monocular depth estimation models (e.g., DepthAnythingV2) to provide high-fidelity depth priors, improving performance on complex geometries and sparse views [S1]. GeoNeRF represents a Transformer-based approach that explicitly reasons about geometric morphology via cost volumes before rendering [S1].

## Sources
[S1] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
