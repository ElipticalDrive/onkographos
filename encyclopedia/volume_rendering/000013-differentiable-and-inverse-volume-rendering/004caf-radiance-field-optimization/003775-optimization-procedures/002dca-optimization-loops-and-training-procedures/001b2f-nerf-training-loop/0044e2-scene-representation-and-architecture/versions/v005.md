# Scene Representation and Architecture

Also called: Radiance Field Approximation, Neural Radiance Field Model

## Overview
Scene Representation and Architecture defines the neural network structure used to approximate the continuous radiance field $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$, where $\mathbf{x}$ is a 3D position, $\mathbf{d}$ is a viewing direction, $\sigma$ is volume density, and $\mathbf{c}$ is view-dependent color [Parent]. It solves the problem of representing complex 3D geometry and appearance in a differentiable format suitable for gradient-based optimization via backpropagation through the volumetric rendering integral [Parent]. It sets itself apart from siblings like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by defining the *model* that predicts physical properties, whereas ray marching defines the *sampling* strategy. It differs from [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md) by focusing on the internal topology of the network rather than the spatial decomposition of the scene.

## Shared principles
The core mechanism is the approximation of the continuous radiance field using a neural network or a collection of networks. The function $F_\Theta$ is differentiable with respect to parameters $\Theta$, allowing gradient-based optimization [Parent, S1, S3].

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
Encompasses specific neural network topologies and architectural modifications (e.g., Transformers, SIRENs) designed to address limitations such as computational cost, generalization, and geometric ambiguity. It differs from siblings by focusing on internal network topology rather than spatial data structures.

### [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md)
Addresses scalability limits by decomposing the 3D scene volume or training data into manageable sub-regions or components using multiple specialized sub-networks. It sets itself apart by focusing on structural decomposition of the scene space rather than the internal topology of a single network.

### [Specific Model Architectures and Extensions](10278-specific-model-architectures-and-extensions.md)
Defines specific neural network topologies and input configurations primarily originating from the NSVF framework, explicitly configuring feature connections between density and texture networks. It differs from the canonical NeRF by supporting various input encodings such as appearance embeddings.

## Background
The canonical MLP-based implicit representation was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [Parent, S1, S3]. This work established the use of positional encoding and coupled density/color networks as the standard baseline for differentiable volume rendering.

## Sources
[S1] Mildenhall, B., Srinivasan, P. P., Ortiz-Cayón, R., Kalantari, N. K., Ramamoorthi, R., Ng, R., & Martin-Brualla, R. (2020). NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis.
[S3] Barron, B. T., Mildenhall, B., Verbin, D., Srinivasan, P. P., & Hedman, P. (2022). Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields.
