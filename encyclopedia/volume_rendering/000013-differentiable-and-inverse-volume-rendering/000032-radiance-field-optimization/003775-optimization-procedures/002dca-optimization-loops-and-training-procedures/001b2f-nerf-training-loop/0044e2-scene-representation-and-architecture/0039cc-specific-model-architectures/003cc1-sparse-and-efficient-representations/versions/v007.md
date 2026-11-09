# Sparse and Efficient Representations

Also called: Sparse NeRF, Sparse Radiance Fields, KiloNeRF, Mega-NeRF, ProNeRF

## Overview
Sparse and Efficient Representations address the computational and memory bottlenecks of global Neural Radiance Fields (NeRFs) by decomposing the scene representation into localized, discrete, or sparse structures. Unlike global parametric functions that query a single deep network for every sample, these methods localize predictions to specific spatial regions, enabling parallel training, reduced memory footprints, and faster inference. They distinguish themselves from siblings like [AdaNeRF](6195-adanerf.md) or [Transformer-based Architectures](8998-transformer-based-architectures.md) by explicitly trading global continuity for spatial sparsity, often utilizing voxel grids, spatial partitions, or projection-aware sampling to skip empty space or reduce sample counts.

## Shared principles
The core mechanism remains the parametric function $F_\Theta$ that maps a 3D position $\mathbf{x} \in \mathbb{R}^3$ and a viewing direction $\mathbf{d} \in \mathbb{R}^3$ to a volume density $\sigma \in \mathbb{R}$ and an RGB color $\mathbf{c} \in \mathbb{R}^3$. This function is implemented as one or more neural networks, typically utilizing positional encoding $\gamma(\mathbf{x})$ to map raw coordinates to a higher-dimensional space using sinusoidal functions at increasing frequencies [Parent].

**Rendering Integration:**
During training, rays are sampled at points $\mathbf{x}_i$ along the ray. The networks predict $\sigma_i$ and $\mathbf{c}_i$. The pixel color $C(\mathbf{r})$ is computed via volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
and $\delta_i$ is the distance between samples [Parent].

**Input Representation:**
Raw coordinates are rarely used directly. They are mapped to a higher-dimensional space using positional encoding $\gamma(\mathbf{x})$. For a coordinate $\mathbf{x}$, the encoding includes the original value and sinusoidal functions at increasing frequencies:
$$
\gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
$$
where $L$ is the number of frequency bands [Parent].

## Subtopics

### [Mega-NeRF Spatial Partitioning](8776-mega-nerf-spatial-partitioning.md)
Mega-NeRF decomposes a large-scale 3D scene into disjoint spatial cells, each managed by an independent neural network submodule, to enable parallel training and inference for virtual fly-throughs. Unlike global NeRF models that suffer from memory bottlenecks and slow convergence on large scenes, this method localizes the prediction of opacity and color to the cell containing the query point, significantly reducing the training dataset size per module. It distinguishes itself from other sparse methods by using a fixed 2D tessellation of centroids rather than dynamic voxelization or hash grids, and by explicitly incorporating appearance embeddings to handle lighting variations across images.

### [Sparse Radiance Field (SRF) Representation](8876-sparse-radiance-field-srf-representation.md)
A Sparse Radiance Field (SRF) is a voxel-based scene representation that encodes density and view-dependent radiance colors into a sparse 3D grid, serving as a discrete, memory-efficient alternative to global neural radiance fields (NeRFs). Unlike global MLP-based representations, SRFs localize predictions to specific spatial regions, enabling direct mesh extraction via Marching Cubes and facilitating large-scale dataset generation for learning-based inverse rendering.

### [KiloNeRF](8942-kilonerf.md)
KiloNeRF is a sparse representation method that decomposes a large, deep Neural Radiance Field (NeRF) into a set of small, shallow MLPs arranged in a 3D grid. It solves the inefficiency of querying a single global network for small spatial portions by using a teacher-student training paradigm where knowledge is distilled from a standard NeRF into these lightweight local models. It is distinguished from siblings like Mega-NeRF by its specific use of a cached sparse 3D grid for surface localization and its reliance on dynamic parameter loading for thousands of tiny networks.

### [ProNeRF](9673-pronerf.md)
ProNeRF is a sparse sampling method for implicit neural radiance fields that uses a Projection-Aware Sampling (PAS) network to predict a small number of high-density 3D points per ray, enabling high-quality rendering with significantly fewer samples than standard NeRF. It achieves an optimal trade-off between memory footprint, speed, and quality, being 15–23× faster than vanilla NeRF while achieving higher PSNR than both NeRF and HyperReel. Unlike generalizable radiance fields that require storing all training images for epipolar projection, ProNeRF uses a fixed, small subset of reference views, ensuring a consistent memory footprint regardless of the number of training views.

### NeRF-XL Multi-GPU Representation Strategy
**Definition:** A parameter distribution strategy for multi-GPU NeRF training where each GPU handles a distinct 3D tiled region with independent hash encoding and density MLP, but shares the color MLP and appearance/camera embeddings across all GPUs [S1].

**Parameters:**
*   **Hash Encoding:** Independent per GPU, confined within a non-overlapping bounding box [S1].
*   **Density MLP:** Independent per GPU [S1].
*   **Color MLP:** Shared across all GPUs using Distributed Data Parallel (DDP) to ensure consistent color prediction for novel views [S1].
*   **Appearance Embedding:** Shared across GPUs; set to zero during novel-view rendering if enabled during training [S1].
*   **Camera Pose Embedding:** Shared across GPUs if camera optimization is enabled [S1].

**Comparison:** Unlike independent training where each GPU has its own full NeRF, this method unifies instantiations as a 'single NeRF' with spatially partitioned parameters, increasing model capacity without redundancy [S1].

## Background
Sparse representations emerged to address the scalability limits of global NeRFs, particularly for large-scale scenes and real-time applications. Early works like KiloNeRF introduced the concept of distilling global knowledge into local shallow networks [S1]. SRFs provided a discrete voxel-based alternative that facilitated mesh extraction [S1]. Mega-NeRF focused on spatial partitioning for large-scale fly-throughs [S1]. ProNeRF introduced projection-aware sampling to reduce the number of required samples per ray [S1]. NeRF-XL extended these concepts to multi-GPU distributed training, ensuring consistency across spatially partitioned parameters [S1].

## Sources
[S1] NeRF-XL: Scaling NeRFs with Multiple GPUs, https://doi.org/10.1007/978-3-031-73195-2_6, fetched 2026-10-26
