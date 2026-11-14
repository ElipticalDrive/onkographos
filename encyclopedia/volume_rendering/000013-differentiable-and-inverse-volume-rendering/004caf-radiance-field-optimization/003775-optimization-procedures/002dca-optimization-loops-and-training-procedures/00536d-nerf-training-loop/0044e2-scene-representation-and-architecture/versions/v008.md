# Scene Representation and Architecture

Also called: Radiance Field Representation, Neural Radiance Field Architecture

## Overview
Scene Representation and Architecture defines the neural network topologies and data structures used to approximate the continuous radiance field $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$ in differentiable volume rendering. It solves the problem of representing complex 3D geometry and appearance as a differentiable function that can be optimized end-to-end via gradient descent. It sets itself apart from its siblings by establishing the canonical MLP-based baseline; [Specific Model Architectures](8585-specific-model-architectures.md) modifies the network topology for speed or generalization, while [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md) decomposes the scene volume for scalability.

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
Replaces or augments standard density/color MLPs with structures like sparse grids, tensor decompositions, or transformers to address training speed, generalization, or dynamic scene modeling. Choose when the canonical MLP is too slow or lacks generalization.

### [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md)
Decomposes the 3D scene volume or training data into independent sub-networks to enable large-scale static scenes and distributed training. Choose when the scene is too large for a single monolithic network.

### [Specific Model Architectures and Extensions](10278-specific-model-architectures-and-extensions.md)
Defines topologies from the NSVF framework, explicitly configuring feature connections between density and texture networks and supporting specific input encodings. Choose when following the NSVF lineage or requiring specific appearance embedding configurations.

## Background
The canonical neural radiance field architecture was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S1]. It established the use of MLPs with positional encoding and skip connections as the standard for implicit 3D representation. Prior work, such as the Neural Scalar Field Volume (NSVF) framework, explored similar neural approximations but with different connectivity patterns [S1, unofficial]. The architecture has since become the foundational baseline for most differentiable volume rendering research.

## Sources
[S1] Mildenhall, B., Srinivasan, P. P., Ortiz-Cayon, R., Kalantari, N. K., Ramamoorthi, R., Ng, R., & Davis, A. (2020). NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis. arXiv preprint arXiv:2003.08934.
