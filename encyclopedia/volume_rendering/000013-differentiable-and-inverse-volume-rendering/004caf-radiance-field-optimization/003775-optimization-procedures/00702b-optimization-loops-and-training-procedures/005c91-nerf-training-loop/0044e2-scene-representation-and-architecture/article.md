# Scene Representation and Architecture


## Overview
Scene Representation and Architecture defines the neural network topologies, data structures, and optimization strategies used to represent and optimize the radiance field $F_\Theta(\mathbf{x}, \mathbf{d}) = (\sigma, \mathbf{c})$ [G]. It sets itself apart from siblings like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) and [Training Configuration](2332-training-configuration.md) by focusing on the underlying network topology and spatial decomposition rather than sampling or optimization hyperparameters [G].

## Shared principles
All methods in this family share the mechanism of mapping 6D input coordinates (3D position $\mathbf{x}$ and 2D viewing direction $\mathbf{d}$) to 4D output values (density $\sigma$ and color $\mathbf{c}$) using a differentiable neural network [G]. This allows the radiance field to be optimized through the volumetric rendering integral [G].

**Input Representation:**
Raw coordinates $\mathbf{x}$ and $\mathbf{d}$ are rarely used directly due to the spectral bias of MLPs, which struggle to fit high-frequency details [G]. They are mapped to a higher-dimensional space using positional encoding $\gamma(\cdot)$ [G]. For a coordinate $\mathbf{x}$, the encoding includes the original value and sinusoidal functions at increasing frequencies:
$$
\gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
$$
where $L$ is the number of frequency bands [G].

**Network Topology:**
The canonical implementation uses two coupled Multi-Layer Perceptrons (MLPs): a density network $F_\sigma$ and a color network $F_c$ [G].
1.  **Density Network:** Takes the encoded position $\gamma(\mathbf{x})$ as input. It consists of $L$ hidden layers with ReLU activations and skip connections from the input to every 4th layer to preserve low-frequency information [G]. The output is the density $\sigma$.
2.  **Color Network:** Takes the encoded position $\gamma(\mathbf{x})$, the raw viewing direction $\mathbf{d}$, and the intermediate feature vector from the density network (usually the output of the last hidden layer before the density head) as input. It predicts the color $\mathbf{c}$ using a final ReLU or sigmoid activation [G].

**Rendering Integration:**
During training, rays are sampled at points $\mathbf{x}_i$ along the ray. The MLPs predict $\sigma_i$ and $\mathbf{c}_i$. The pixel color $C(\mathbf{r})$ is computed via volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
and $\delta_i$ is the distance between samples [G].

## Subtopics

### [Specific Model Architectures](8585-specific-model-architectures.md)
Defines neural network topologies and optimization strategies for the radiance field. Sets itself apart by modifying network topology (e.g., multi-space decomposition, tensor grids) or introducing auxiliary networks for tasks like depth estimation or instance segmentation. Choose when computational cost, geometric ambiguity, or dynamic content handling requires architectural changes beyond the standard MLP.

### [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md)
Addresses scalability limits by decomposing the scene into multiple smaller representations optimized independently. Sets itself apart by focusing on structural decomposition of the scene space rather than internal network topology. Choose for parallel training, reduced memory footprint, or dynamic content handling.

### [Specific Model Architectures and Extensions](10278-specific-model-architectures-and-extensions.md)
Represents a scene as a single MLP mapping 3D position and viewing direction to color and density. Sets itself apart by using a monolithic MLP without spatial partitioning or explicit geometric priors, relying on positional encoding and hierarchical sampling. Choose for standard unbounded scene representation within defined ray bounds.

### [NeRFReN](11981-nerfren.md)
Separates the radiance field into transmitted and reflected components to handle reflective surfaces like glass and mirrors. Sets itself apart by explicitly modeling the reflection fraction derived from transmitted geometry. Choose when rendering reflective materials is required.

### [AR-NeRF](12493-ar-nerf.md)
Employs an aperture camera model to represent defocus effects and viewpoint changes. Sets itself apart by using a collection of rays converging at a focus plane per pixel, enabling unsupervised learning of depth and defocus. Choose for single-image defocus and depth estimation.

### [Neural Implicit Surfaces with PBR](13481-neural-implicit-surfaces-with-pbr.md)
Integrates a Signed Distance Function (SDF) with a Physically-Based Rendering (PBR) material model. Sets itself apart by deriving smooth normals from the SDF gradient and using a precomputed BRDF LUT for specular calculation. Choose for accurate mirror-like reflections and geometric smoothness.

### [Human and Avatar Modeling](18785-human-and-avatar-modeling.md)
Specializes in synthesizing novel views of dynamic human avatars from sparse inputs. Sets itself apart by combining NeRF volume rendering with deformable human body models (e.g., SMPL) to handle articulated poses. Choose for dynamic human rendering and pose interpolation.

#### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function [S1]. It incorporates an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one [S1]. The density distribution is defined as $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where $s$ is a scaling parameter and $x$ is the SDF or occupancy value [S1].

#### NELF
A neural volumetric rendering framework modeling scene appearance using light transport vectors, enabling realistic relighting and view synthesis of human portraits from only five input images [S2]. It uses a UNet-style CNN to extract per-view features, an MLP to regress volume density and transport vectors, and estimates environment maps to disentangle lighting [S2]. Trained on synthetic data and adapted to real images via a domain adaptation module, it achieves photo-realistic, lighting-consistent renderings [S2].

#### Nerfacto Pipeline
The default baseline model in Nerfstudio, integrating several advances from recent NeRF literature into a single, efficient pipeline for in-the-wild reconstruction [S3, unofficial]. It combines a hash-grid encoding for compact and expressive scene representation, Mip-NeRF 360 proposal sampling for efficient ray sampling, distortion loss from Mip-NeRF 360 to stabilize convergence, scene contraction to handle unbounded scenes, camera poses refinement, and appearance embeddings to account for illumination variations across input images [S3, unofficial].

## Background
The canonical NeRF architecture was introduced by Mildenhall et al. in 2020 [G]. It established the paradigm of using a single MLP to map 6D coordinates to radiance properties [G]. Subsequent developments addressed specific limitations: reflective surfaces (NeRFReN, Neural Implicit Surfaces with PBR) [S1], dynamic humans (Neural Body, NeuMan) [S2], defocus (AR-NeRF) [S1], and scalability (Scene Representation and Partitioning, Nerfacto) [S3, unofficial]. The field has evolved from monolithic MLPs to hybrid representations and specialized architectures for specific domains like human avatars and reflective materials [G].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S3] From reconstruction to reasoning : neural radiance fields for comprehensive 3D scene understanding, https://theses.hal.science/tel-05693528v1, fetched 2026-10-26 (unofficial)
