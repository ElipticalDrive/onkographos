# Sparse and Efficient Representations


## Overview
Sparse and Efficient Representations address the prohibitive memory and computational costs of dense, monolithic Neural Radiance Fields (NeRFs) by decomposing the scene representation into smaller, localized, or discrete structures. This family enables the rendering of large-scale scenes or high-fidelity details that would otherwise exceed hardware limits. Unlike the canonical NeRF which uses a single global MLP for the entire volume, these methods partition the scene spatially (via grids, voxels, or subnetworks) or reduce the sampling density, allowing for parallel training, faster inference, and reduced memory footprints.

## Shared principles
The canonical representation maps a 3D position $\mathbf{x} \in \mathbb{R}^3$ and a viewing direction $\mathbf{d} \in \mathbb{R}^3$ to a volume density $\sigma \in \mathbb{R}$ and an RGB color $\mathbf{c} \in \mathbb{R}^3$ using two coupled MLPs: a density network $F_\sigma$ and a color network $F_c$.

**Positional Encoding:**
Raw coordinates are mapped to higher dimensions using positional encoding $\gamma(\mathbf{x})$. For a coordinate $\mathbf{x}$, the encoding includes the original value and sinusoidal functions at increasing frequencies:
$$
\gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
$$
where $L$ is the number of frequency bands. This allows the MLP to learn high-frequency details [S1][S3].

**Network Topology:**
The density network $F_\sigma$ takes the encoded position $\gamma(\mathbf{x})$ as input. It typically consists of $L$ hidden layers with ReLU activations and skip connections from the input to every 4th layer to preserve low-frequency information [S1][S3]. The output is the density $\sigma$.

The color network $F_c$ takes the encoded position $\gamma(\mathbf{x})$, the raw viewing direction $\mathbf{d}$, and the intermediate feature vector from the density network (usually the output of the last hidden layer before the density head) as input. It predicts the color $\mathbf{c}$ using a final ReLU or sigmoid activation [S1][S3].

**Rendering Integration:**
During training, rays are sampled at points $\mathbf{x}_i$ along the ray. The MLPs predict $\sigma_i$ and $\mathbf{c}_i$. The pixel color $C(\mathbf{r})$ is computed via volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
and $\delta_i$ is the distance between samples [S1][S3].

## Subtopics

### [Mega-NeRF Spatial Partitioning](8776-mega-nerf-spatial-partitioning.md)
Mega-NeRF decomposes a large-scale scene into spatial cells, each managed by a separate MLP submodule, to enable parallel training and reduce memory consumption. It distinguishes itself from monolithic NeRFs by using a fixed 2D grid of centroids to assign pixels to local subnetworks, and from other sparse methods by explicitly pruning rays based on inferred geometry to minimize redundant computation.

### [Sparse Radiance Field (SRF) Representation](8876-sparse-radiance-field-srf-representation.md)
A Sparse Radiance Field (SRF) is a voxel-based scene representation that stores density and view-dependent radiance directly in a sparse 3D grid, serving as a discrete, efficient alternative to the continuous Multi-Layer Perceptron (MLP) used in standard NeRFs. It sets itself apart from dense voxel methods by storing features only for occupied regions in COO format, and from MLP-based methods by enabling direct mesh extraction via Marching Cubes and faster optimization through sparse data structures.

### [KiloNeRF](8942-kilonerf.md)
KiloNeRF is a method that decomposes a large, deep Neural Radiance Field (NeRF) into a set of small, shallow NeRFs arranged in a 3D grid, using a teacher-student training paradigm to distill knowledge from a standard NeRF. It sets itself apart from siblings like Mega-NeRF by explicitly optimizing for inference speed through the use of thousands of tiny MLPs and a cached sparse grid for surface localization, achieving acceleration of up to three orders of magnitude compared to the original NeRF.

### [ProNeRF](9673-pronerf.md)
ProNeRF is a few-point volumetric rendering method that uses a Projection-Aware Sampling (PAS) network to predict sparse 3D sampling points and opacity adjustments, allowing high-quality rendering with significantly fewer samples than standard NeRF. It achieves an optimal trade-off between memory footprint, speed, and quality, being 15–23× faster than vanilla NeRF while achieving higher PSNR than both NeRF and HyperReel. Unlike generalizable radiance fields that store all training images for epipolar projection, ProNeRF uses a fixed, small subset of reference views, ensuring a consistent memory footprint regardless of the number of training views.

### NeRF-XL Multi-GPU Representation Strategy
**Definition**
A parameter distribution strategy for multi-GPU NeRF training where each GPU handles a distinct 3D tiled region with independent hash encoding and density MLP, but shares the color MLP and appearance/camera embeddings across all GPUs [S1].

**Parameters**
*   **Hash Encoding:** Independent per GPU, confined within a non-overlapping bounding box [S1].
*   **Density MLP:** Independent per GPU [S1].
*   **Color MLP:** Shared across all GPUs using Distributed Data Parallel (DDP) to ensure consistent color prediction for novel views [S1].
*   **Appearance Embedding:** Shared across GPUs; set to zero during novel-view rendering if enabled during training [S1].
*   **Camera Pose Embedding:** Shared across GPUs if camera optimization is enabled [S1].

**Comparison**
Unlike independent training where each GPU has its own full NeRF, this method unifies instantiations as a 'single NeRF' with spatially partitioned parameters, increasing model capacity without redundancy [S1].

## Background
The concept of sparse and efficient representations in NeRFs emerged as a response to the $O(N)$ memory and computational complexity of dense voxel grids and large MLPs, which limited NeRFs to small scenes or low resolutions. Mega-NeRF [S1] introduced spatial partitioning to scale to large environments. Sparse Radiance Fields (SPARF) [S1] explored discrete voxel grids with sparsity to enable mesh extraction. KiloNeRF [S1] focused on inference speed via knowledge distillation into tiny MLPs. ProNeRF [S1] addressed sampling efficiency by predicting optimal sample locations. NeRF-XL [S1] provided a framework for scaling these representations across multiple GPUs.

## Sources
[S1] NeRF-XL: Scaling NeRFs with Multiple GPUs, https://doi.org/10.1007/978-3-031-73195-2_6, fetched 2026-10-26
