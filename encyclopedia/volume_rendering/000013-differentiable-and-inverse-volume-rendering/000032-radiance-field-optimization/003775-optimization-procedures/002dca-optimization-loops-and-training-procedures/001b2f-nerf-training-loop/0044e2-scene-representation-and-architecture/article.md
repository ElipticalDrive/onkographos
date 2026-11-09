# Scene Representation and Architecture
Also called: NeRF Architecture, Radiance Field Model Structure
## Overview
This article defines the canonical implicit volumetric representation used in Neural Radiance Fields (NeRF), establishing the standard Multi-Layer Perceptron (MLP) topology, positional encoding schemes, and volume rendering integration that serve as the baseline for optimization [S1, S3]. It addresses the problem of how to map 3D spatial coordinates and viewing directions to density and color values to synthesize novel views. It sets itself apart from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the neural network's internal structure and feature encoding rather than the sampling strategy, and from [Specific Model Architectures](8585-specific-model-architectures.md) by describing the unmodified baseline rather than variants like VaxNeRF or SimpleNeRF.
## Shared principles
The canonical representation maps a 3D position $\mathbf{x} \in \mathbb{R}^3$ and a viewing direction $\mathbf{d} \in \mathbb{R}^3$ to a volume density $\sigma \in \mathbb{R}$ and an RGB color $\mathbf{c} \in \mathbb{R}^3$ using two coupled MLPs: a density network $F_\sigma$ and a color network $F_c$.

**Positional Encoding:**
Raw coordinates are mapped to higher dimensions using positional encoding $\gamma(\mathbf{x})$. For a coordinate $\mathbf{x}$, the encoding includes the original value and sinusoidal functions at increasing frequencies:
$$
\gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
$$
where $L$ is the number of frequency bands. This allows the MLP to learn high-frequency details [S1, S3].

**Network Topology:**
The density network $F_\sigma$ takes the encoded position $\gamma(\mathbf{x})$ as input. It typically consists of $L$ hidden layers with ReLU activations and skip connections from the input to every 4th layer to preserve low-frequency information [S1, S3]. The output is the density $\sigma$.

The color network $F_c$ takes the encoded position $\gamma(\mathbf{x})$, the raw viewing direction $\mathbf{d}$, and the intermediate feature vector from the density network (usually the output of the last hidden layer before the density head) as input. It predicts the color $\mathbf{c}$ using a final ReLU or sigmoid activation [S1, S3].

**Rendering Integration:**
During training, rays are sampled at points $\mathbf{x}_i$ along the ray. The MLPs predict $\sigma_i$ and $\mathbf{c}_i$. The pixel color $C(\mathbf{r})$ is computed via volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
and $\delta_i$ is the distance between samples [S1, S3].

## Subtopics

### [Specific Model Architectures](8585-specific-model-architectures.md)
Modifies the canonical MLP structure, sampling, or data structures (e.g., hash grids, visual hulls) to address speed, sparse-view, or dynamic scene limitations. Choose when the baseline NeRF is too slow or fails on sparse inputs.

### [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md)
Decomposes the scene volume into disjoint sub-volumes, each managed by a local MLP or grid. Choose for large-scale scenes where global memory consumption is prohibitive.

### [Specific Model Architectures and Extensions](10278-specific-model-architectures-and-extensions.md)
Defines specific configurations for density and texture networks within Neural Scene Vector Fields (NSVF) and related models, focusing on feature concatenation and topology. Choose when configuring NSVF-specific parameter structures.

## Background
The canonical NeRF architecture was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S1, S3]. It established the standard use of positional encoding and coupled MLPs for implicit 3D representation. The method is widely used as a baseline in novel view synthesis, 3D reconstruction, and downstream tasks like robotics and AR [S1, S3].

## Sources
[S1] Mildenhall, B., Srinivasan, P. P., Ortiz-Cayon, R., Kalantari, N. K., Ramamoorthi, R., Ng, R., & Davis, A. (2020). NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis. arXiv:2003.08934.
[S2] LiDAR-NeRF Implementation Notes.
[S3] Neural Confidence Field (ICON) Implementation Notes.
[S4] SimpleNeRF Implementation Notes.
[S10] MERF Implementation Notes.
