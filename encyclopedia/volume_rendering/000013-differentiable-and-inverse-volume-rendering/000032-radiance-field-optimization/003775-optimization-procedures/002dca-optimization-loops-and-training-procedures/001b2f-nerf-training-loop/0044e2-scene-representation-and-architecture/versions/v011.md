# Scene Representation and Architecture

Also called: Radiance Field Models, Neural Radiance Fields (NeRF) Architectures

## Overview
Scene Representation and Architecture defines the neural network structures used to approximate the continuous radiance field $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$ in differentiable volume rendering. It solves the problem of representing 3D geometry and appearance as a differentiable function that can be optimized via gradient descent. It sets itself apart from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the network topology and input encoding rather than the sampling strategy, and from [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md) by typically assuming a single global representation rather than decomposing the scene.

## Shared principles
All methods in this family share the mechanism of mapping 6D input coordinates (3D position $\mathbf{x}$ and 2D viewing direction $\mathbf{d}$) to 4D output values (density $\sigma$ and color $\mathbf{c}$) using a differentiable neural network. This allows the radiance field to be optimized through the volumetric rendering integral.

**Input Representation:**
Raw coordinates $\mathbf{x}$ and $\mathbf{d}$ are rarely used directly due to the spectral bias of MLPs, which struggle to fit high-frequency details. They are mapped to a higher-dimensional space using positional encoding $\gamma(\cdot)$ [S1][S3]. For a coordinate $\mathbf{x}$, the encoding includes the original value and sinusoidal functions at increasing frequencies:
$$
\gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
$$
where $L$ is the number of frequency bands [S1][S3].

**Network Topology:**
The canonical implementation uses two coupled Multi-Layer Perceptrons (MLPs): a density network $F_\sigma$ and a color network $F_c$ [S1][S3].
1.  **Density Network:** Takes the encoded position $\gamma(\mathbf{x})$ as input. It consists of $L$ hidden layers with ReLU activations and skip connections from the input to every 4th layer to preserve low-frequency information [S1][S3]. The output is the density $\sigma$.
2.  **Color Network:** Takes the encoded position $\gamma(\mathbf{x})$, the raw viewing direction $\mathbf{d}$, and the intermediate feature vector from the density network (usually the output of the last hidden layer before the density head) as input. It predicts the color $\mathbf{c}$ using a final ReLU or sigmoid activation [S1][S3].

**Rendering Integration:**
During training, rays are sampled at points $\mathbf{x}_i$ along the ray. The MLPs predict $\sigma_i$ and $\mathbf{c}_i$. The pixel color $C(\mathbf{r})$ is computed via volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
and $\delta_i$ is the distance between samples [S1][S3].

## Subtopics

### [Specific Model Architectures](8585-specific-model-architectures.md)
Extends the canonical MLP with modifications to topology, input representation, or optimization loops to address generalization, single-view synthesis, and high-resolution rendering.

### [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md)
Decomposes the scene into multiple sub-regions or local MLPs to address scalability and distributed training, aggregating contributions across partition boundaries.

### [Specific Model Architectures and Extensions](10278-specific-model-architectures-and-extensions.md)
Defines topologies from the NSVF framework, explicitly configuring feature connections between density and texture networks and supporting specific input encodings.

### [NeRFReN](11981-nerfren.md)
Separates the radiance field into transmitted and reflected components to handle reflective surfaces like glass and mirrors, modeling a reflection fraction $\beta$ derived from transmitted geometry.

### [Neural Body](12176-neural-body.md)
Combines NeRF with a vertex-based deformable human body model (SMPL), anchoring latent variables to mesh vertices to model articulated, moving bodies.
#### MobileNeRF
- **Definition:** A framework training a NeRF-like model based on a polygonal mesh with color, feature, and opacity MLPs attached to each mesh vertex [S1].
- **Step:** Discretize alpha values [S1]. Super-sample features for anti-aliasing [S1]. During rendering, rasterize the mesh with associated features and opacities according to the viewing position [S1]. Use a small MLP to shade each pixel [S1].
- **Layout:** Polygonal mesh with color, feature, and opacity MLPs attached to each mesh vertex [S1].
- **Number:** Speed approximately 10 times faster than SNeRG [S1].

## Background
The canonical Neural Radiance Field (NeRF) architecture was introduced by Mildenhall et al. in 2020 [S1]. It established the standard of using coupled MLPs with positional encoding to represent continuous radiance fields. Subsequent developments, such as MobileNeRF (June 2022) and EfficientNeRF (July 2022), extended this representation to mesh-based and tree-based structures for efficiency [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
