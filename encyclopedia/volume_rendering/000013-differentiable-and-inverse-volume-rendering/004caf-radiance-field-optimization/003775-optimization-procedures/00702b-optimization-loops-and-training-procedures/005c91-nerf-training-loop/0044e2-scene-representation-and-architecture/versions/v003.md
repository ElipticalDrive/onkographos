# Scene Representation and Architecture

Also called: Radiance Field Models, Neural Radiance Field Architectures

## Overview
Scene Representation and Architecture defines the parametric function $F_\Theta$ that maps 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ to volume density $\sigma$ and color $\mathbf{c}$, serving as the core learnable prior in radiance-field optimization [Parent]. It differs from sibling methods by focusing exclusively on the network topology, input encoding, and spatial decomposition strategies that define *what* is being optimized, whereas siblings like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) define *how* it is evaluated, and [Training Configuration](2332-training-configuration.md) defines the hyperparameters.

## Shared principles
The fundamental mechanism is the approximation of the continuous radiance field using a neural network or a collection of networks. The function $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$ is differentiable with respect to $\Theta$, allowing gradient-based optimization via backpropagation through the volumetric rendering integral [Parent].

**Input Representation:**
Raw coordinates $\mathbf{x}$ and $\mathbf{d}$ are rarely used directly due to the spectral bias of MLPs, which struggle to fit high-frequency details. They are mapped to a higher-dimensional space using positional encoding $\gamma(\cdot)$ [Parent, S1, S3]. For a coordinate $\mathbf{x}$, the encoding includes the original value and sinusoidal functions at increasing frequencies:
$$
\gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
$$
where $L$ is the number of frequency bands [Parent, S1, S3].

**Network Topology:**
The canonical implementation uses two coupled Multi-Layer Perceptrons (MLPs): a density network $F_\sigma$ and a color network $F_c$ [Parent, S1, S3].
1.  **Density Network:** Takes the encoded position $\gamma(\mathbf{x})$ as input. It consists of $L$ hidden layers with ReLU activations and skip connections from the input to every 4th layer to preserve low-frequency information [Parent, S1, S3]. The output is the density $\sigma$.
2.  **Color Network:** Takes the encoded position $\gamma(\mathbf{x})$, the raw viewing direction $\mathbf{d}$, and the intermediate feature vector from the density network (usually the output of the last hidden layer before the density head) as input. It predicts the color $\mathbf{c}$ using a final ReLU or sigmoid activation [Parent, S1, S3].

**Rendering Integration:**
During training, rays are sampled at points $\mathbf{x}_i$ along the ray. The MLPs predict $\sigma_i$ and $\mathbf{c}_i$. The pixel color $C(\mathbf{r})$ is computed via volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
and $\delta_i$ is the distance between samples [Parent, S1, S3].

## Subtopics

### [Specific Model Architectures](8585-specific-model-architectures.md)
Encompasses specialized neural network topologies (e.g., sparse grids, transformers) designed to address real-time constraints, single-view ambiguity, or dynamic scenes. Choose when standard MLPs are too slow or inaccurate for the target application.

### [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md)
Decomposes the scene volume into independent sub-volumes, each modeled by a separate local network. Choose for large-scale scenes requiring scalability or distributed training.

### [Specific Model Architectures and Extensions](10278-specific-model-architectures-and-extensions.md)
Defines configurations from the NSVF framework, focusing on input representations and explicit appearance embeddings. Choose when handling appearance variations or unbounded spaces with established MLP priors.

## Background
The canonical MLP-based architecture was introduced in the original NeRF paper [Parent]. It established the standard two-MLP structure with positional encoding, which became the baseline for subsequent architectural variations [Parent, S1, S3].

## Sources
[S1] Specific Model Architectures and Extensions, https://example.com/10278, 2023-10-01
[S3] Scene Representation and Partitioning, https://example.com/9444, 2023-10-01
