# Scene Representation and Architecture
Also called: Radiance Field Network Topology, Neural Volume Representation

## Overview
This family defines the neural network structures used to approximate the continuous radiance field $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$ in differentiable volume rendering. It solves the problem of representing complex 3D geometry and appearance as a differentiable function that can be optimized via gradient descent through the volumetric rendering integral. It sets itself apart from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the *function approximation* (the MLP or grid structure) rather than the *sampling strategy*, and from [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md) by covering the fundamental single-network baseline before decomposition.

## Shared principles
All methods in this family share the mechanism of mapping 6D input coordinates (3D position $\mathbf{x}$ and 2D viewing direction $\mathbf{d}$) to 4D output values (density $\sigma$ and color $\mathbf{c}$) using a differentiable neural network. This allows the radiance field to be optimized through the volumetric rendering integral.

**Input Representation:**
Raw coordinates $\mathbf{x}$ and $\mathbf{d}$ are rarely used directly due to the spectral bias of MLPs, which struggle to fit high-frequency details. They are mapped to a higher-dimensional space using positional encoding $\gamma(\cdot)$ [S1, unofficial][S3]. For a coordinate $\mathbf{x}$, the encoding includes the original value and sinusoidal functions at increasing frequencies:
$$
\gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
$$
where $L$ is the number of frequency bands [S1, unofficial][S3].

**Network Topology:**
The canonical implementation uses two coupled Multi-Layer Perceptrons (MLPs): a density network $F_\sigma$ and a color network $F_c$ [S1, unofficial][S3].
1.  **Density Network:** Takes the encoded position $\gamma(\mathbf{x})$ as input. It consists of $L$ hidden layers with ReLU activations and skip connections from the input to every 4th layer to preserve low-frequency information [S1, unofficial][S3]. The output is the density $\sigma$.
2.  **Color Network:** Takes the encoded position $\gamma(\mathbf{x})$, the raw viewing direction $\mathbf{d}$, and the intermediate feature vector from the density network (usually the output of the last hidden layer before the density head) as input. It predicts the color $\mathbf{c}$ using a final ReLU or sigmoid activation [S1, unofficial][S3].

**Rendering Integration:**
During training, rays are sampled at points $\mathbf{x}_i$ along the ray. The MLPs predict $\sigma_i$ and $\mathbf{c}_i$. The pixel color $C(\mathbf{r})$ is computed via volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
and $\delta_i$ is the distance between samples [S1, unofficial][S3].

## Subtopics

### [Specific Model Architectures](8585-specific-model-architectures.md)
Defines the standard MLP-based mapping with positional encoding and skip connections. Choose this for the canonical NeRF baseline.

### [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md)
Decomposes the scene into multiple sub-regions or distinct neural fields to address scalability or dynamic content. Choose this for large-scale or distributed training.

### [Specific Model Architectures and Extensions](10278-specific-model-architectures-and-extensions.md)
Defines specific neural network topologies and input configurations originating from the NSVF framework, focusing on feature connections between density and texture networks. Choose this for variants requiring specific appearance embedding configurations.

## Background
The canonical architecture was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S1]. It was developed to overcome the limitations of explicit 3D representations (like meshes) and implicit scalar fields (like SDFs) by jointly modeling geometry and view-dependent appearance. The use of positional encoding was motivated by the need to help MLPs learn high-frequency details, a technique previously used in Fourier Features for networks (FFNets) [S3].

## Sources
[S1] Mildenhall, B., Srinivasan, P. P., Ortiz-Cayon, R., Kalantari, N. K., Ramamoorthi, R., Ng, R., & Davis, A. (2020). NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis. arXiv preprint arXiv:2003.08934.
[S3] Tancik, M., Srinivasan, P. P., Mildenhall, B., Fridovich-Keil, S., Raghavan, N., Singhal, U., ... & Ng, R. (2020). Fourier Features Let Networks Learn High Frequency Functions in Low Dimensional Domains. Advances in Neural Information Processing Systems, 33, 7462-7471.
