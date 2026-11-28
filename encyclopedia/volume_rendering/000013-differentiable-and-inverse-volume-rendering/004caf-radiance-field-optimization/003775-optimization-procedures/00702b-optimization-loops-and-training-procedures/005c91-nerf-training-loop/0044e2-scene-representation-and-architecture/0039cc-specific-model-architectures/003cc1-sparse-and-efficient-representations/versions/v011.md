# Sparse and Efficient Representations


## Overview
Sparse and Efficient Representations address the computational and memory bottlenecks of dense, monolithic Neural Radiance Fields (NeRFs) by decomposing the scene representation into smaller, localized, or structured components. This family enables scalable training for large-scale scenes and faster inference through spatial partitioning, explicit sparse grids, or generalized 2D projections. Unlike the canonical NeRF which uses a single deep MLP for all queries, these methods distribute the representational capacity across multiple sub-networks, voxel structures, or planar features to reduce redundancy and accelerate convergence [S1, S2].

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
Decomposes large-scale scenes into spatial cells with independent MLP submodules, enabling parallel training and data pruning based on visibility.

### [MultiPlaneNeRF](8875-multiplanenerf.md)
Replaces implicit encoding with a non-trainable 2D image-based representation, allowing generalization to new objects without decoder retraining.

### [Sparse Radiance Field (SRF) Representation](8876-sparse-radiance-field-srf-representation.md)
Uses an explicit, sparse 3D grid in COO format to store non-empty voxels, avoiding hierarchical tree structures.

### [KiloNeRF](8942-kilonerf.md)
Replaces a single large MLP with thousands of small, independent MLPs on a regular 3D grid, accelerated by an occupancy grid and empty space skipping.

### [ProNeRF](9673-pronerf.md)
A few-point volumetric rendering method using a Projection-Aware Sampling network to predict sparse 3D points, reducing sample count while maintaining quality.

### [Quadri-Plane Radiance Fields](11128-quadri-plane-radiance-fields.md)
Represents scenes using four 2D feature planes (three spatial, one directional) coupled with a lightweight decoder, decoupling spatial and directional information.

### NeRF-XL Multi-GPU Representation Strategy
- **Definition:** A parameter distribution strategy for multi-GPU NeRF training where each GPU handles a distinct 3D tiled region with independent hash encoding and density MLP, but shares the color MLP and appearance/camera embeddings across all GPUs [S1].
- **Parameters:** Hash Encoding is independent per GPU, confined within a non-overlapping bounding box. Density MLP is independent per GPU. Color MLP is shared across all GPUs using Distributed Data Parallel (DDP) to ensure consistent color prediction for novel views. Appearance Embedding is shared across GPUs; set to zero during novel-view rendering if enabled during training. Camera Pose Embedding is shared across GPUs if camera optimization is enabled [S1].
- **Comparison:** Unlike independent training where each GPU has its own full NeRF, this method unifies instantiations as a 'single NeRF' with spatially partitioned parameters, increasing model capacity without redundancy [S1].

## Background
The concept of decomposing NeRF representations for efficiency originated with KiloNeRF, which demonstrated that replacing a single large MLP with thousands of small MLPs could accelerate rendering by three orders of magnitude without sacrificing visual quality [S2]. Subsequent work introduced sparse voxel grids (SRF) to handle irregular volumetric data more efficiently than dense grids [S1]. Mega-NeRF extended spatial partitioning to large-scale scenes by assigning distinct MLP weights to geometric regions and pruning training data based on visibility [S1]. MultiPlaneNeRF shifted focus from implicit encoding to explicit 2D image projections to improve generalization [S1]. ProNeRF introduced projection-aware sampling to reduce the number of required samples [S1]. Quadri-Plane Radiance Fields leveraged 2D feature planes to enable efficient super-resolution [S1]. NeRF-XL provided a framework for scaling these representations across multiple GPUs by sharing specific components like the color MLP while partitioning others [S1].

## Sources
[S1] NeRF-XL: Scaling NeRFs with Multiple GPUs, https://doi.org/10.1007/978-3-031-73195-2_6, fetched 2026-10-26
