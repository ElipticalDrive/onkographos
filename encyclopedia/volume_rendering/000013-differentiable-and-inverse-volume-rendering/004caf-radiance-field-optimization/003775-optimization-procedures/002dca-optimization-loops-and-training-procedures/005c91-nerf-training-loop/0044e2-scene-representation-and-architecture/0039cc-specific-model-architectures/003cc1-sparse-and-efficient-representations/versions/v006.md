# Sparse and Efficient Representations

Also called: Sparse NeRF, Large-scale NeRF, Distributed NeRF

## Overview
Sparse and Efficient Representations address the computational and memory bottlenecks of dense Neural Radiance Fields (NeRFs) in large-scale or high-resolution scenes. While the parent [Specific Model Architectures](8585-specific-model-architectures.md) relies on a monolithic MLP to map 3D coordinates to density and color, sparse methods decompose the scene representation into localized substructures—such as spatial partitions, voxel grids, or distributed parameters—to enable parallel processing and reduced memory footprint. This family sets itself apart from dense alternatives by explicitly managing spatial locality or hardware distribution, allowing for the rendering of scenes that would otherwise exceed GPU memory limits or require prohibitive training times.

## Shared principles
The core principle of this family is the decomposition of the global scene representation into manageable, localized components. Unlike the canonical NeRF which uses a single global function $F(\mathbf{x}, \mathbf{d})$, sparse methods ensure that the prediction for a point $\mathbf{x}$ depends only on a specific subset of parameters associated with the spatial region containing $\mathbf{x}$.

This localization is achieved through one of three primary mechanisms:
1.  **Spatial Partitioning:** The 3D space is divided into disjoint or overlapping cells (e.g., Mega-NeRF). Each cell contains its own MLP or data structure. Querying a point involves identifying the containing cell and invoking its local model.
2.  **Voxelization:** The scene is discretized into a 3D grid (e.g., SRF, KiloNeRF). Only non-empty or active voxels store data, often using sparse data structures (like COO format) to save memory.
3.  **Parameter Distribution:** In multi-GPU setups, the model parameters themselves are partitioned across devices (e.g., NeRF-XL). Each GPU manages a spatial region with independent density/hash parameters but shares color prediction parameters to maintain consistency.

All methods retain the standard volumetric rendering equation for compositing the local predictions into a final pixel color:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
The key difference lies in how $\sigma_i$ and $\mathbf{c}_i$ are retrieved: they are no longer outputs of a single global MLP but are derived from the local structure associated with sample point $\mathbf{x}_i$.

## Subtopics

### [Mega-NeRF Spatial Partitioning](8776-mega-nerf-spatial-partitioning.md)
Mega-NeRF decomposes large-scale scenes into a fixed top-down 2D grid of spatial cells, each managed by a separate MLP submodule. It is chosen for virtual fly-throughs of large outdoor scenes where altitude variance is low. It differs from siblings by using a 2D centroid grid with fixed height and appearance embeddings for lighting, rather than 3D voxels or hash grids.

### [Sparse Radiance Field (SRF) Representation](8876-sparse-radiance-field-srf-representation.md)
SRF replaces dense MLPs with a discrete, sparse 3D voxel grid storing density and radiance directly. It is chosen when direct mesh extraction (via Marching Cubes) or multi-resolution handling is required. It differs from KiloNeRF by storing explicit voxel features rather than distilling knowledge into tiny MLPs, and from Mega-NeRF by using a regular grid structure rather than a centroid-based partition.

### [KiloNeRF](8942-kilonerf.md)
KiloNeRF accelerates rendering by decomposing a large NeRF into a set of small, shallow MLPs arranged in a 3D grid, using a teacher-student distillation paradigm. It is chosen for real-time rendering of large scenes. It differs from Mega-NeRF by using a 3D grid and distillation, and from SRF by using neural submodules rather than raw voxel values.

### [ProNeRF](9673-pronerf.md)
ProNeRF achieves high-quality rendering with significantly fewer samples than vanilla NeRF using a Projection-Aware Sampling (PAS) network. It is chosen for generalizable radiance fields where memory footprint must remain constant regardless of training view count. It differs from the other sparse methods by focusing on sample efficiency and cross-view projection rather than spatial partitioning of the scene representation itself.

### NeRF-XL Multi-GPU Representation Strategy
NeRF-XL is a parameter distribution strategy for multi-GPU NeRF training. Each GPU handles a distinct 3D tiled region with independent hash encoding and density MLP, but shares the color MLP and appearance/camera embeddings across all GPUs [S1]. This ensures consistent color prediction for novel views, which would otherwise be inconsistent if each GPU maintained its own independent color MLP [S1]. The method unifies these instantiations as a "single NeRF" with spatially partitioned parameters, increasing model capacity without redundancy [S1]. Hash encoding and density MLPs are confined within non-overlapping bounding boxes per GPU, while the color MLP uses Distributed Data Parallel (DDP) sharing [S1]. Appearance embeddings are shared; a zero embedding is used for novel-view rendering if enabled during training [S1].

## Background
The term "Sparse and Efficient Representations" in the context of NeRFs emerged as a response to the $O(N)$ memory and computational complexity of dense MLP-based NeRFs, particularly for large-scale urban environments or high-resolution indoor scans. Early works like KiloNeRF [S2, S3, unofficial] and Mega-NeRF [S1] demonstrated that scene representation could be localized. The concept of multi-GPU scaling, specifically NeRF-XL [S1], formalized the distribution of these sparse parameters across hardware to enable training on scenes with billions of points. These methods collectively shifted the paradigm from monolithic neural fields to distributed, sparse, or hybrid representations.

## Sources
[S1] NeRF-XL: Scaling NeRFs with Multiple GPUs, https://doi.org/10.1007/978-3-031-73195-2_6, fetched 2026-10-26
