# Non-Baked Models

Also called: Hybrid Representations, Learning separate scene features from MLP parameters

## Overview
Non-baked models refer to scene representations that decouple the learning of geometric/appearance features from the neural network parameters, typically using hybrid structures where explicit features are stored in memory and queried by smaller, faster MLPs [S1]. This approach solves the trade-off between the high memory cost of fully implicit MLPs and the limited expressiveness of purely explicit grids, enabling smaller MLPs for improved training and inference speed at the cost of increased memory usage [S1]. Unlike baked models, which pre-compute and store the final MLP outputs in accessible data structures to accelerate inference without affecting training time, non-baked models retain the differentiability of the feature-to-output mapping during the optimization loop [S1].

## Shared principles
All methods in this family share the mechanism of mapping 6D input coordinates to 4D output values using a differentiable neural network, allowing the radiance field to be optimized end-to-end through the volumetric rendering integral.

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

### [AdaNeRF](6195-adanerf.md)
Adapts the representation dynamically.

### [SinNeRF](6199-sinnerf.md)
Uses sinusoidal activations.

### [NeRF-RPN](8663-nerf-rpn.md)
Uses region proposal networks.

### [RefSR-NeRF](8691-refsr-nerf.md)
Focuses on reflection and super-resolution.

### [AligNeRF](8695-alignerf.md)
Improves alignment.

### [DiffusioNeRF](8703-diffusionerf.md)
Integrates diffusion models.

### [MultiPlaneNeRF](8875-multiplanenerf.md)
Uses multi-plane representations.

### [Pool proposer architecture](8953-pool-proposer-architecture.md)
Uses pooling mechanisms.

### [Transformer-based Architectures](8998-transformer-based-architectures.md)
Replaces MLPs with Transformers.

### [MS-NeRF](9111-ms-nerf.md)
Multi-scale representations.

### [NeRDi](9112-nerdi.md)
Depth-integrated representation.

### [Neural Light Field Representations](9273-neural-light-field-representations.md)
Light field variants.

### [RING-NeRF](9323-ring-nerf.md)
Ring-based acquisition handling.

### [Sparse and Efficient Representations](9326-sparse-and-efficient-representations.md)
Optimizes sparsity.

### [Geometric Priors and Depth Integration](9395-geometric-priors-and-depth-integration.md)
Uses depth priors.

### [Generative and Style-based Models](9572-generative-and-style-based-models.md)
Generative approaches.

### [IRON](10251-iron.md)
Implicit Octree Radiance Fields.

### [pi-GAN](10298-pi-gan.md)
GAN-based implicit fields.

### [Instance-NeRF](10611-instance-nerf.md)
Instance-specific fields.

### [MIMO-NeRF](10928-mimo-nerf.md)
Multi-input multi-output.

### [Appearance and Lighting Decomposition](11061-appearance-and-lighting-decomposition.md)
Separates lighting.

### [Quadri-Plane Radiance Fields](11128-quadri-plane-radiance-fields.md)
Quadri-plane structures.

### [GNeuVox](11357-gneuvox.md)
GPU-optimized neural voxels.

### [HexPlane](11358-hexplane.md)
Hex-plane representations.

### [STensoRRF](11359-stensorrf.md)
Tensor-based radiance fields.

### [GIRAFFE](11360-giraffe.md)
Generative Inverse Radiance Fields.

### [Ced-NeRF](11364-ced-nerf.md)
Conditional editing.

### [Dynamic and Deformable Fields](11366-dynamic-and-deformable-fields.md)
Handles dynamic scenes.

## Background
The term "Non-Baked Models" originates from the classification of NeRF optimization procedures, specifically distinguishing between methods that pre-compute outputs ("baked") and those that learn separate scene features from MLP parameters ("non-baked") [S1]. This category emerged as a response to the memory bottlenecks of standard MLP-based NeRFs, aiming to improve both training and inference speed by reducing the MLP size, albeit with increased memory overhead for storing the explicit features [S1]. The concept is closely related to hybrid representations, which combine implicit neural networks with explicit data structures, and includes methods that may omit neural networks entirely in favor of purely explicit scene representations, though these are included due to their relevance and similarity to NeRF [S1]. Common techniques associated with this family include ray termination, space skipping, and hierarchical sampling using coarse and fine MLPs [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
