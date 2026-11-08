# VastGaussian

Also called: Block-Based Partitioning, Large-Scale 3DGS via Overlapping Blocks

## Overview
VastGaussian is a divide-and-conquer method for rendering massive 3D scenes that exceed the memory and computational limits of a single 3D Gaussian Splatting (3DGS) model [S1, S2, unofficial]. It partitions the scene into spatially-disjoint or overlapping blocks, trains independent 3DGS models for each block, and merges the results at rendering time using feathered overlap to handle boundaries [S1, S2, unofficial]. It sets itself apart from monolithic 3DGS by enabling scalability to city-scale environments, distinguishing itself from sibling methods like [GSPull](2773-gspull.md) (which focuses on pull-based rendering) and [Sparse-Controlled 3DGS](6145-sparse-controlled-3d-gs.md) (which focuses on sparsity) by explicitly managing scene partitioning and block-level independence [S1, S2, unofficial].

## How it works
**Inputs:** A large-scale scene dataset (e.g., aerial or urban imagery with camera poses) and a partitioning strategy (overlapping blocks or spatially-disjoint regions) [S1, S2, unofficial].
**Outputs:** A set of independent 3DGS models (one per block) and a merging strategy for rendering [S1, S2, unofficial].

**Data Structures:**
The method does not define a new primitive data structure but utilizes the standard 3DGS Gaussian representation (position, covariance, opacity, spherical harmonics coefficients) within each block [S1, unofficial]. The scene is represented as a collection of block-specific Gaussian sets $\{G_1, G_2, ..., G_N\}$ [S1, unofficial].

**Steps:**
1.  **Partitioning:** Divide the scene into $N$ blocks. Sources describe two variations:
    *   *Overlapping Blocks:* Blocks overlap spatially to ensure seamless blending [S1, unofficial].
    *   *Spatially-Disjoint Regions:* Blocks are non-overlapping, requiring specific handling for boundaries and lighting variations [S2, unofficial].
2.  **Independent Training:** Train a separate 3DGS model for each block using only the training images and Gaussians relevant to that block [S1, S2, unofficial].
    *   *Appearance Handling:* For disjoint regions, decouple appearance handling to account for varying lighting conditions across blocks [S2, unofficial].
3.  **Merging at Rendering:** When rendering a view:
    *   Identify all blocks that intersect the view frustum or the region of interest [S1, S2, unofficial].
    *   Render each active block independently to produce intermediate color/depth buffers [S1, S2, unofficial].
    *   **Feathered Overlap:** If blocks overlap, blend the rendered outputs from adjacent blocks using a feathering (blending) function at the boundaries to eliminate seams [S1, S2, unofficial]. For disjoint blocks, this step is omitted or replaced by hard cuts, though [S2, unofficial] implies merging strategies are still applied for consistency.

**Parameters:**
*   **Block Size/Overlap:** Not given in the sources.
*   **Feathering Width:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Seams:** In overlapping blocks, improper feathering can cause visible seams or double-rendering artifacts [S1, S2, unofficial].
*   **Lighting Inconsistency:** In disjoint regions, global lighting changes may cause color mismatches between adjacent blocks if appearance is not decoupled or normalized [S2, unofficial].
*   **Memory Management:** Each block requires separate memory allocation for its Gaussian set; managing $N$ independent models can increase CPU overhead [S1, S2, unofficial].

## Subtopics

### Progressive Partitioning for Large-Scale 3DGS
A strategy to divide a large scene into multiple cells where training cameras and 3D Gaussians are properly distributed [S3]. It distributes training cameras and 3D Gaussians using an airspace-aware visibility criterion to ensure balanced load and coverage [S3].

### Large Scene Reconstruction via Chunking
Partitioning a 3D Gaussian Splatting scene into manageable chunks to handle physical memory limits and scalability challenges in large scenes [S4].
*   **Camera-Based Chunking:** Lin et al. split scenes into chunks based on the distribution of camera positions and the respective Structure from Motion (SfM) reconstruction [S4].
*   **Uniform Chunking:** Liu et al. split scenes into uniform chunks and prune chunks with low contribution to rendering [S4].
*   **Skybox Background:** Kerbl et al. define a low-resolution skybox on a sphere surrounding each chunk, defining the surrounding scene as background [S4].
*   **Distributed Training:** Zhao et al. distribute Gaussians across multiple GPUs, assigning pixels to be rendered to GPUs in tiles, and transferring the Gaussians between them as needed [S4]. Dynamic load balancing can make distributed training efficient and scale to up to 32 GPUs [S4].

## Operating characteristics
*   **Time Complexity:** Training time is reduced per block due to smaller scene subsets, but total training time includes the sum of all blocks plus merging overhead [S1, S2, unofficial]. Rendering time scales with the number of active blocks, but each block renders fewer Gaussians [S1, S2, unofficial].
*   **Memory Complexity:** Memory usage is distributed across blocks, allowing reconstruction of scenes larger than single-GPU memory limits [S1, S2, unofficial]. However, storing $N$ independent models increases total storage compared to a single optimized model [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Enables reconstruction of city-scale and large-scale scenes; improves scalability; allows parallel training [S1, S2, unofficial][S4].
*   **Limitations:** Potential for seams at block boundaries; increased complexity in merging and management; lighting inconsistencies in disjoint regions [S1, S2, unofficial].
*   **When to Use:** Large-scale scenes (city-scale, aerial) that cannot fit into a single 3DGS model's memory or training budget [S1, S2, unofficial].
*   **When Not to Use:** Small to medium scenes where monolithic 3DGS is sufficient and simpler to manage [S1, S2, unofficial].

## Background
VastGaussian was introduced in CVPR'24 [S2, unofficial]. It is part of the broader effort to scale 3D Gaussian Splatting to large environments, building on the foundational 3DGS method [S1, S2, unofficial]. It is often compared with CityGaussian and CityGaussianV2, which also use hierarchical or block-based approaches but may differ in LoD strategies and geometry regularization [S2, unofficial]. The method addresses the lack of huge datasets and memory constraints for 3DGS scalability [S4].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S4] Virtual memory for 3D Gaussian Splatting, https://zenodo.org/records/19690968, fetched 2026-10-26
