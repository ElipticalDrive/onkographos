# Scene Representation and Architecture

Also called: Radiance Field Approximation, Neural Implicit Representation

## Overview
Scene Representation and Architecture defines the neural network topology and input encoding strategies used to approximate the continuous radiance field $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$ within the NeRF training loop. It solves the problem of representing a 3D scene as a differentiable function that can be optimized via gradient descent to match observed 2D images. It sets itself apart from [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by defining *what* is being predicted (the field parameters) rather than *how* the prediction is sampled and integrated, and from [Specific Model Architectures](8585-specific-model-architectures.md) by establishing the canonical baseline against which specialized extensions are compared.

## Shared principles
The family shares the mechanism of mapping 6D input coordinates (3D position $\mathbf{x}$ and 2D viewing direction $\mathbf{d}$) to 4D output values (scalar density $\sigma$ and 3D color $\mathbf{c}$) using a differentiable neural network. This allows the radiance field to be optimized end-to-end through the volumetric rendering integral.

**Input Representation:**
Raw coordinates $\mathbf{x}$ and $\mathbf{d}$ are rarely used directly due to the spectral bias of MLPs, which struggle to fit high-frequency details. They are mapped to a higher-dimensional space using positional encoding $\gamma(\cdot)$ [S1, S3]. For a coordinate $\mathbf{x}$, the encoding includes the original value and sinusoidal functions at increasing frequencies:
$$
\gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
$$
where $L$ is the number of frequency bands [S1, S3].

**Network Topology:**
The canonical implementation uses two coupled Multi-Layer Perceptrons (MLPs): a density network $F_\sigma$ and a color network $F_c$ [S1, S3].
1.  **Density Network:** Takes the encoded position $\gamma(\mathbf{x})$ as input. It consists of $L$ hidden layers with ReLU activations and skip connections from the input to every 4th layer to preserve low-frequency information [S1, S3]. The output is the density $\sigma$.
2.  **Color Network:** Takes the encoded position $\gamma(\mathbf{x})$, the raw viewing direction $\mathbf{d}$, and the intermediate feature vector from the density network (usually the output of the last hidden layer before the density head) as input. It predicts the color $\mathbf{c}$ using a final ReLU or sigmoid activation [S1, S3].

**Rendering Integration:**
During training, rays are sampled at points $\mathbf{x}_i$ along the ray. The MLPs predict $\sigma_i$ and $\mathbf{c}_i$. The pixel color $C(\mathbf{r})$ is computed via volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
and $\delta_i$ is the distance between samples [S1, S3].

## Subtopics

### [Specific Model Architectures](8585-specific-model-architectures.md)
Encompasses specific neural network topologies and architectural modifications used to approximate the continuous radiance field. It differs from the canonical approach by replacing or augmenting the monolithic density-color MLP with specialized components like voxel grids, transformers, or multi-branch decoders to address limitations in training speed, generalization, or dynamic content handling.

### [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md)
Addresses scalability limits by decomposing the 3D scene volume or training data into manageable sub-regions or components. It sets itself apart by focusing on the structural decomposition of the scene space (spatial or content-based) rather than the internal topology of a single network, enabling distributed training and large-scale scene representation.

### [Specific Model Architectures and Extensions](10278-specific-model-architectures-and-extensions.md)
Defines specific neural network topologies and input configurations for approximating radiance fields, primarily originating from the NSVF framework. It sets itself apart by explicitly configuring feature connections between density and texture networks and supporting various input encodings to handle different scene constraints.

## Background
The canonical scene representation was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S1]. It established the use of a fully connected MLP with positional encoding to implicitly represent 3D scenes as continuous functions of density and color. This approach replaced explicit 3D representations like voxels or meshes, which suffered from resolution limits or topological complexity, with a differentiable implicit representation that could be optimized directly from 2D images. The method has become the foundational baseline for subsequent research in neural radiance fields.

## Sources
[S1] Mildenhall, B., Srinivasan, P. P., Ortiz-Cayon, R., Kalantari, N. K., Ramamoorthi, R., Ng, R., & Davis, A. (2020). NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis. arXiv preprint arXiv:2003.08934.
[S3] Barron, B. T., Mildenhall, B., Verbin, D., Srinivasan, P. P., & Hedman, P. (2021). Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields. arXiv preprint arXiv:2103.13415.
