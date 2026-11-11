# Scene Representation and Architecture

Also called: Neural Radiance Field Architectures, Radiance Field Models

## Overview
This family defines the neural network structures and data representations used to approximate the continuous volumetric radiance field $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$ within the NeRF Training Loop [970-nerf-training-loop.md]. It solves the problem of how to parameterize scene geometry and appearance for differentiable optimization. It is distinguished from sibling methods by focusing on the internal topology of the scene model itself, rather than the sampling strategy [2199-training-ray-marching-and-integration.md], hyperparameters [2332-training-configuration.md], or loss functions [8586-regularization-and-loss-functions.md].

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
Specific Model Architectures refers to the family of neural network structures designed to represent and optimize radiance fields, extending the canonical Multi-Layer Perceptron (MLP) baseline to address limitations in generalization, efficiency, dynamic scenes, and generative capabilities. While the canonical NeRF [Parent] uses a single global MLP to map 6D coordinates to density and color, this family encompasses methods that replace or augment this structure with sparse grids, transformers, diffusion priors, or multi-branch decoders to solve problems such as slow training, poor generalization to unseen scenes, inability to handle dynamic content, and lack of semantic control. Key distinctions include the shift from per-scene optimization to generalizable learning (e.g., [Transformer-based Architectures](8998-transformer-based-architectures.md)), the decomposition of geometry and appearance for relighting ([Appearance and Lighting Decomposition](11061-appearance-and-lighting-decomposition.md)), and the integration of external priors for single-view or sparse-view reconstruction ([SinNeRF](6199-sinnerf.md), [NeRDi](9112-nerdi.md)).

### [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md)
This family encompasses methods that decompose a 3D scene into multiple sub-regions or distinct neural fields to address scalability, dynamic content, or distributed training constraints. Unlike the canonical single-network NeRF, these methods partition the scene representation to enable parallel processing, handle large-scale environments, or separate static and dynamic elements. Key distinctions include MegaNeRF's use of spatial Voronoi cells for data parallelism in large-scale training [8581-meganerf], NeRFPlayer's decomposition into stationary, deformation, and radiance fields for dynamic streaming [9440-nerfplayer], and the distributed spatial partitioning method's use of geometry-based PDFs for balanced GPU workload distribution [10332-spatial-partitioning-for-distributed-nerf].

### [Specific Model Architectures and Extensions](10278-specific-model-architectures-and-extensions.md)
This family defines specific neural network topologies and input configurations for approximating radiance fields, primarily originating from the NSVF framework. It sets itself apart from the canonical [NeRF Training Loop](970-nerf-training-loop.md) by explicitly configuring the feature connections between density and texture networks, and by supporting various input encodings (such as appearance embeddings or specific coordinate subsets) to handle different scene constraints.

### [NeRFReN](11981-nerfren.md)
NeRFReN addresses the failure of standard Neural Radiance Fields (NeRF) to render reflective surfaces, such as glass and mirrors, by separating the radiance field into transmitted and reflected components. It distinguishes itself from sibling methods by explicitly modeling the reflection fraction $\beta$ derived from the transmitted geometry, allowing for the separation of reflection artifacts from physical entities.

### [Neural Body](12176-neural-body.md)
Neural Body is a method for synthesizing novel views of dynamic human avatars from sparse multi-view videos by combining NeRF volume rendering with a vertex-based deformable human body model (SMPL). It solves the problem of modeling articulated, moving bodies, which standard NeRF approaches struggle with due to pose changes. By anchoring latent variables to the vertices of a deformable mesh, it enables interpolation between frames and poses, establishing a foundational paradigm for neural field rendering of human avatars.

### NELF
- [S1] A neural volumetric rendering framework modeling scene appearance using light transport vectors, enabling realistic relighting and view synthesis of human portraits from only five input images.
- [S1] A UNet-style CNN extracts per-view features.
- [S1] An MLP regresses volume density and transport vectors.
- [S1] Environment maps are estimated to disentangle lighting.
- [S1] Trained on synthetic data and adapted to real images via a domain adaptation module.

### NeRF with Per-Frame Latent Codes
- [S1] A method using per-frame learned latent codes and adding facial expression as a 76-dimensional coefficient of a morphable model constructed from Face2Face [210].

### MobileNeRF
- [S1] A framework training a NeRF-like model based on a polygonal mesh with color, feature, and opacity MLPs attached to each mesh vertex.
- [S1] Discretize alpha values.
- [S1] Super-sample features for anti-aliasing.
- [S1] During rendering, rasterize the mesh with associated features and opacities according to the viewing position.
- [S1] Use a small MLP to shade each pixel.
- [S1] Polygonal mesh with color, feature, and opacity MLPs attached to each mesh vertex.
- [S1] Speed approximately 10 times faster than SNeRG.

## Background
The canonical NeRF architecture was introduced by Mildenhall et al. in 2020 [S1]. It established the use of a global MLP with positional encoding and skip connections as the standard representation. Subsequent developments, such as MobileNeRF (June 2022) [S1], EfficientNeRF (July 2022) [S1], Nerfies (December 2020) [S1], NerFace (December 2020) [S1], HyperNeRF (June 2021) [S1], NELF (July 2021) [S1], and CoNeRF (December 2021) [S1], have expanded this foundation to address specific constraints like efficiency, dynamic content, and relighting.

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
