# Sparse and Efficient Representations

Also called: Sparse NeRF, Efficient NeRF, Grid-based NeRF, Voxel-based NeRF

## Overview
Sparse and Efficient Representations address the prohibitive memory and computational costs of dense, global neural radiance fields (NeRFs) by decomposing the scene representation into localized, sparse, or low-dimensional structures. This family enables the training and rendering of large-scale scenes or high-resolution details that would otherwise exceed hardware limits. Unlike monolithic NeRFs that use a single global MLP for the entire volume, these methods partition the scene spatially, replace implicit MLPs with explicit sparse grids, or decouple spatial and directional information into 2D planes, thereby reducing memory complexity from $O(N^3)$ to $O(N)$ or $O(N^2)$ and accelerating inference by orders of magnitude.

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

### [Mega-NeRF Spatial Partitioning](8776-mega-nerf-spatial-partitioning.md)
Decomposes large scenes into spatial cells with independent MLPs, enabling parallel training and data pruning.

### [MultiPlaneNeRF](8875-multiplanenerf.md)
Replaces implicit encoding with non-trainable 2D image projections, enabling generalization to unseen objects without decoder retraining.

### [Sparse Radiance Field (SRF) Representation](8876-sparse-radiance-field-srf-representation.md)
Uses an explicit, sparse 3D grid in COO format to store density and radiance, avoiding hierarchical tree structures.

### [KiloNeRF](8942-kilonerf.md)
Distills a global teacher NeRF into a grid of small, shallow student MLPs, accelerating rendering by up to three orders of magnitude.

### [ProNeRF](9673-pronerf.md)
Uses a Projection-Aware Sampling network to predict sparse 3D points and opacity adjustments, enabling high-quality rendering with significantly fewer samples.

### [Quadri-Plane Radiance Fields](11128-quadri-plane-radiance-fields.md)
Represents scenes using four 2D feature planes (three spatial, one directional) coupled with a lightweight decoder, decoupling spatial and directional information.

### NeRF-XL Multi-GPU Representation Strategy
**Definition:** A parameter distribution strategy for multi-GPU NeRF training where each GPU handles a distinct 3D tiled region with independent hash encoding and density MLP, but shares the color MLP and appearance/camera embeddings across all GPUs [S1].
**Parameters:**
*   **Hash Encoding:** Independent per GPU, confined within a non-overlapping bounding box [S1].
*   **Density MLP:** Independent per GPU [S1].
*   **Color MLP:** Shared across all GPUs using Distributed Data Parallel (DDP) to ensure consistent color prediction for novel views, as independent color MLPs would lead to inconsistent predictions for the same view direction and appearance embedding [S1].
*   **Appearance Embedding:** Shared across GPUs; set to zero during novel-view rendering if enabled during training [S1].
*   **Camera Pose Embedding:** Shared across GPUs if camera optimization is enabled [S1].
**Comparison:** Unlike independent training where each GPU has its own full NeRF, this method unifies instantiations as a 'single NeRF' with spatially partitioned parameters, increasing model capacity without redundancy [S1].

## Background
The concept of sparse representations in volume rendering originated from the need to scale NeRFs to large-scale scenes. Mega-NeRF introduced spatial partitioning for virtual fly-throughs [S1]. KiloNeRF demonstrated the efficiency of decomposing deep NeRFs into shallow local networks [S1][S2, S3]. Sparse Radiance Fields (SRF) explored explicit voxel storage [S1]. MultiPlaneNeRF focused on generalization via image-based representations [S1]. ProNeRF addressed inference speed via adaptive sampling [S1]. Quadri-Plane Radiance Fields leveraged 2D super-resolution techniques for efficient rendering [S1]. NeRF-XL extended these concepts to multi-GPU distributed training [S1].

## Sources
[S1] NeRF-XL: Scaling NeRFs with Multiple GPUs, https://doi.org/10.1007/978-3-031-73195-2_6, fetched 2026-10-26
