# Scene Representation and Architecture

Also called: NeRF Architectures, Radiance Field Models

## Overview
Scene Representation and Architecture defines the neural network structures that map 6D input coordinates (3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$) to radiance field properties (density $\sigma$ and color $\mathbf{c}$) [Parent]. While the canonical NeRF uses a simple two-MLP structure, this family encompasses modifications to the network topology, input representations, and output heads to address specific challenges such as single-view synthesis, dynamic scenes, high-resolution rendering, and semantic editing. These architectures differ from their siblings by introducing specialized components like diffusion priors, multi-scale grids, or physical material decompositions to enhance reconstruction quality, efficiency, or controllability [S1][S2, unofficial].

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
Specific Model Architectures refers to the family of neural network designs that map 6D input coordinates (3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$) to radiance field properties (density $\sigma$ and color $\mathbf{c}$) [Parent]. While the canonical NeRF uses a simple two-MLP structure, this family encompasses modifications to the network topology, input representations, and output heads to address specific challenges such as single-view synthesis, dynamic scenes, high-resolution rendering, and semantic editing. These architectures differ from their siblings by introducing specialized components like diffusion priors, multi-scale grids, or physical material decompositions to enhance reconstruction quality, efficiency, or controllability.

### [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md)
This family of methods addresses the scalability limits of single-network NeRFs by decomposing the scene into multiple, smaller representations (submodules, blocks, or fields) that are optimized and queried independently. While the parent [Scene Representation and Architecture](11368-scene-representation-and-architecture.md) establishes the global mapping from 6D coordinates to radiance properties, these methods partition the domain to enable parallel training, reduce memory footprint, or handle dynamic content. They differ from siblings like [Specific Model Architectures](8585-specific-model-architectures.md) by focusing on the structural decomposition of the scene space rather than the internal topology of the neural network itself.

### [Specific Model Architectures and Extensions](10278-specific-model-architectures-and-extensions.md)
This family encompasses the foundational neural radiance field architectures that map 6D input coordinates to density and color, serving as the baseline for subsequent optimizations and extensions. It solves the problem of implicit scene representation by learning a continuous volumetric function from multi-view images. It sets itself apart from its siblings by using a single, monolithic Multi-Layer Perceptron (MLP) without spatial partitioning or explicit geometric priors, relying entirely on positional encoding and hierarchical sampling to capture scene details. Unlike [Mip-NeRF 360](3306-mip-nerf-360.md) which handles unbounded scenes via contraction, or [NeRF-FF](8669-nerf-ff.md) which addresses defocus, the original architecture assumes a bounded, in-focus scene.

### [NeRFReN](11981-nerfren.md)
NeRFReN addresses the failure of standard Neural Radiance Fields (NeRF) to render reflective surfaces, such as glass and mirrors, by separating the radiance field into transmitted and reflected components. It distinguishes itself from sibling methods by explicitly modeling the reflection fraction $\beta$ derived from the transmitted geometry, allowing for the separation of reflection artifacts from physical entities.

### [MobileNeRF](12170-mobilenerf.md)
MobileNeRF is a framework that trains a NeRF-like model based on a polygonal mesh with color, feature, and opacity MLPs attached to each mesh vertex. It is designed to render novel images efficiently on mobile devices such as phones and light laptops. It sets itself apart from standard volumetric NeRFs by replacing the ray-marching integration with a rasterization-based approach, achieving a speed approximately 10 times faster than SNeRG.

### [Neural Body](12176-neural-body.md)
Neural Body is a method for synthesizing novel views of dynamic human avatars from sparse multi-view videos by combining NeRF volume rendering with a vertex-based deformable human body model (SMPL). It solves the problem of rendering articulated bodies with moving poses, which standard NeRF approaches struggle with due to the complexity of modeling large non-rigid deformations. By anchoring latent variables to the vertices of a deformable mesh, Neural Body enables the interpolation of poses and frames, establishing a foundational paradigm for neural field rendering of human avatars.

### [NeuMan](12480-neuman.md)
NeuMan renders a human together with a scene with novel human poses and novel views from a single video by jointly optimizing a scene NeRF and a pose-driven human NeRF. It sets itself apart from standard NeRF variants by introducing a canonical space based on the Da-pose SMPL mesh to avoid volume collision during warping, and an error-correction network to handle details not expressed by the SMPL model. Unlike methods that require multi-view video or motion capture, NeuMan operates on a single monocular video sequence.

### [AR-NeRF](12493-ar-nerf.md)
AR-NeRF is an extension of Neural Radiance Fields (NeRF) that employs an aperture camera model instead of a pinhole camera model to represent defocus effects and viewpoint changes in a unified manner within a common ray-tracing framework. It enables fully unsupervised learning of depth and defocus effects from single natural images by jointly leveraging defocus and viewpoint cues without explicit supervision or pretrained models. Unlike standard NeRF, which assumes a pinhole camera, AR-NeRF represents each pixel using a collection of rays converging at a focus plane, allowing the model to disentangle defocus-aware and defocus-independent representations through aperture randomized training.

#### NELF
- [S1] A neural volumetric rendering framework modeling scene appearance using light transport vectors, enabling realistic relighting and view synthesis of human portraits from only five input images.
- [S1] A UNet-style CNN extracts per-view features.
- [S1] An MLP regresses volume density and transport vectors.
- [S1] Environment maps are estimated to disentangle lighting.
- [S1] Trained on synthetic data and adapted to real images via a domain adaptation module.

#### NeRF with Per-Frame Latent Codes
- [S1] A method using per-frame learned latent codes and adding facial expression as a 76-dimensional coefficient of a morphable model constructed from Face2Face [210].

#### Nerfacto Pipeline
- [S2, unofficial] The default baseline model in Nerfstudio, integrating several advances from recent NeRF literature into a single, efficient pipeline for in-the-wild reconstruction.
- [S2, unofficial] Combine a hash-grid encoding for compact and expressive scene representation.
- [S2, unofficial] Use Mip-NeRF 360 proposal sampling for efficient ray sampling.
- [S2, unofficial] Apply distortion loss from Mip-NeRF 360 to stabilize convergence.
- [S2, unofficial] Apply scene contraction to handle unbounded scenes.
- [S2, unofficial] Apply camera poses refinement.
- [S2, unofficial] Apply appearance embeddings to account for illumination variations across input images.

## Background
The canonical NeRF architecture was introduced by Mildenhall et al. in 2020 [S1, S3]. It established the paradigm of using coupled MLPs with positional encoding to represent continuous volumetric scenes. Subsequent developments have focused on extending this representation to handle dynamic content (e.g., Neural Body, NeuMan), reflective surfaces (NeRFReN), mobile efficiency (MobileNeRF), and defocus (AR-NeRF) [S1][S2, unofficial]. The Nerfacto pipeline represents a modern aggregation of these advances into a standardized baseline [S2, unofficial].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] From reconstruction to reasoning : neural radiance fields for comprehensive 3D scene understanding, https://theses.hal.science/tel-05693528v1, fetched 2026-10-26 (unofficial)
