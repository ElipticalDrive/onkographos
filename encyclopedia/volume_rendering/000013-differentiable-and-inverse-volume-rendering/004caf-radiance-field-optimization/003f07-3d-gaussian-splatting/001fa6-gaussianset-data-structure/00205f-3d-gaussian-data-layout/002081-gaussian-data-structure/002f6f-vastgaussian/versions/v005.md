# VastGaussian

Also called: Block-Based Partitioning, Large-Scale 3DGS via Block Partitioning

## Overview
VastGaussian is a large-scale 3D Gaussian Splatting method that partitions a scene into spatially-disjoint regions to enable independent training and decoupled appearance handling for varying lighting conditions, merging results at render time with feathered overlap [S2, unofficial]. It distinguishes itself from standard 3DGS by solving the scalability and lighting inconsistency problems in city-scale reconstruction through a divide-and-conquer strategy that isolates local geometry and appearance [S1, S2, unofficial].

## How it works
**Inputs:** A set of input images with known camera poses (SfM reconstruction) and an initial sparse point cloud.
**Outputs:** A set of 3D Gaussian models, one per partition block, stored in a hierarchical structure.

**Data Structures:**
The scene is partitioned into non-overlapping spatial blocks. Each block contains:
1.  A local set of 3D Gaussians (position, covariance, opacity, spherical harmonics coefficients).
2.  A local skybox or background model to handle occlusion from outside the block [S4].
3.  Metadata defining the block's spatial bounds and overlap region for merging.

**Steps:**
1.  **Partitioning:** Divide the scene into spatially-disjoint blocks. The partitioning can be based on camera distribution, uniform grid, or SfM structure [S2, unofficial][S4].
2.  **Local Training:** Train an independent 3D Gaussian Splatting model for each block. During training, the block is treated as an isolated scene.
3.  **Appearance Decoupling:** Handle varying lighting conditions independently per block. This allows different blocks to have different exposure or lighting assumptions without global optimization conflicts [S2, unofficial].
4.  **Background Modeling:** For each block, define a background model (e.g., a low-resolution skybox on a sphere surrounding the chunk) to represent the scene outside the block's immediate vicinity [S4].
5.  **Merging:** At render time, combine the contributions from all blocks. Use feathered blending in the overlapping regions between adjacent blocks to hide seams [S2, unofficial].

**Parameters:**
*   **Block Size:** Not explicitly defined in sources; depends on scene scale and memory limits [S4].
*   **Overlap Width:** Not explicitly defined; sufficient to allow feathered blending [S2, unofficial].
*   **Max GPUs:** Distributed training can scale to up to 32 GPUs [S4].

**Edge Cases:**
*   **Seams:** Poor merging can cause visible seams. Feathered overlap is required [S2, unofficial].
*   **Lighting Discontinuities:** If lighting changes abruptly across block boundaries, decoupled appearance handling is critical [S2, unofficial].
*   **Memory Limits:** Partitioning is necessary when the scene exceeds physical memory [S4].

## Subtopics

### [Page Assignment and Merging](8355-page-assignment-and-merging.md)
Page Assignment and Merging is a spatial partitioning strategy used in VastGaussian to organize millions of 3D Gaussians into fixed-size memory pages for efficient virtual memory management and rendering. It solves the problem of mapping unstructured Gaussian clusters to a structured, mesh-aligned page hierarchy, ensuring that Gaussians are grouped by spatial locality while respecting hardware page size constraints. Unlike standard grid-based voxelization, this method derives its topology from a proxy triangular mesh, allowing it to handle surface-aligned geometry with varying lighting and appearance. It distinguishes itself by combining mesh-based subdivision with greedy merging and explicit page linking to handle overlapping ellipsoids that cross page boundaries.

### [Similarity Guided Partition](14399-similarity-guided-partition.md)
Similarity Guided Partition is a graph-based clustering algorithm that groups 3D Gaussians into spatially-disjoint blocks of bounded size based on spatial proximity and appearance similarity. It solves the problem of inefficient joint pruning by ensuring that potentially redundant Gaussians are evaluated together within the same optimization block. It sets itself apart from naive spatial partitioning by incorporating a scale-aware affinity metric that accounts for both geometric overlap and appearance consistency, and from sibling methods like Page Assignment and Merging by using a parallel greedy growth strategy on a sparse affinity graph rather than page-based memory management.

### Progressive Partitioning for Large-Scale 3DGS
Progressive Partitioning is a strategy to divide a large scene into multiple cells where training cameras and 3D Gaussians are properly distributed [S3]. It distributes training cameras and 3D Gaussians using an airspace-aware visibility criterion [S3].

### Large Scene Reconstruction via Chunking
Large Scene Reconstruction via Chunking partitions a 3D Gaussian Splatting scene into manageable chunks to handle physical memory limits and scalability challenges in large scenes [S4]. Lin et al. split scenes into chunks based on the distribution of camera positions and the respective Structure from Motion (SfM) reconstruction [S4]. Liu et al. split scenes into uniform chunks and prune chunks with low contribution to rendering [S4]. Kerbl et al. define a low-resolution skybox on a sphere surrounding each chunk, defining the surrounding scene as background [S4]. Zhao et al. distribute Gaussians across multiple GPUs, assigning pixels to be rendered to GPUs in tiles, and transferring the Gaussians between them as needed [S4]. The maximum GPU count for distributed training is 32 GPUs [S4].

## Operating characteristics
*   **Scalability:** Enables reconstruction of city-scale scenes (square kilometers) by breaking them into manageable blocks [S2, unofficial][S4].
*   **Memory:** Reduces per-GPU memory footprint by training on chunks [S4].
*   **Performance:** Distributed training can scale to up to 32 GPUs [S4].
*   **Quality:** Decoupled appearance handling improves visual consistency in scenes with varying lighting [S2, unofficial].
*   **Limitations:** Requires careful merging to avoid seams [S2, unofficial]. Complexity increases with the number of blocks [S1, unofficial].

## Background
VastGaussian was introduced in CVPR'24 [S2, unofficial]. It addresses the lack of large-scale datasets and the scalability challenges of 3D Gaussian Splatting in massive scenes [S2, unofficial][S4]. It builds on the divide-and-conquer approach used in neural radiance fields [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S4] Virtual memory for 3D Gaussian Splatting, https://zenodo.org/records/19690968, fetched 2026-10-26
