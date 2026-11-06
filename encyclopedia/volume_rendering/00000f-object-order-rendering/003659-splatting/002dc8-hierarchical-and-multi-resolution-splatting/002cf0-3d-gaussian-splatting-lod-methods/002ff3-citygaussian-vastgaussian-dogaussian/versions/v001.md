# CityGaussian / VastGaussian / DoGaussian

Also called: City-Gaussian, CityGaussianV2, DoGaussian

## Overview
CityGaussian, VastGaussian, and DoGaussian are block-based partitioning methods that enable the training and rendering of large-scale urban scenes by dividing the environment into spatially disjoint or overlapping tiles, training independent 3D Gaussian Splatting (3DGS) models per tile, and merging them at rendering time [S1, S2, unofficial]. They solve the memory and scalability limits of monolithic 3DGS by decoupling scene complexity from GPU memory constraints [S1, S2, unofficial]. Unlike sibling methods that rely on hierarchical data structures like octrees or voxel scaffolds (e.g., [Octree-GS](5441-octree-gs.md), [Scaffold-GS](5442-scaffold-gs.md)), these methods use explicit spatial tiling with overlap handling to manage boundary artifacts and lighting variations [S1, S2, unofficial].

## How it works
The method relies on a divide-and-conquer strategy where the scene is partitioned, trained locally, and merged globally.

### Inputs and Outputs
*   **Inputs:** A set of calibrated images covering a large area (city-scale); camera poses; initial 3DGS parameters (or point cloud initialization).
*   **Outputs:** A set of per-block 3DGS models (Gaussians with position $\mathbf{p}$, covariance $\Sigma$, opacity $\alpha$, spherical harmonic coefficients $\mathbf{c}$) and a merge configuration defining overlap regions and blending weights.

### Data Structures
Each block $B_i$ stores its own 3DGS representation:
*   **Positions:** $\mathbf{p} \in \mathbb{R}^3$
*   **Covariance:** $\Sigma \in \mathbb{R}^{3 \times 3}$ (or rotation $\mathbf{q}$ and scale $\mathbf{s}$)
*   **Opacity:** $\alpha \in [0, 1]$
*   **Color:** Spherical Harmonic coefficients $\mathbf{c}_{l,m}$ for degree $L$.
*   **Block Metadata:** Bounding box $\mathcal{B}_i$, overlap region definitions, and blending masks.

### Steps
1.  **Partitioning:** Divide the scene into spatially disjoint or overlapping blocks (tiles) [S1, S2, unofficial]. VastGaussian uses spatially-disjoint regions, while CityGaussian uses overlapping blocks [S1, S2, unofficial].
2.  **Independent Training:** Train a separate 3DGS model for each block using only the images that fall within or significantly overlap the block's region [S1, S2, unofficial].
    *   *VastGaussian Specific:* Handle varying lighting conditions across the city by decoupling appearance handling per block or using global illumination priors if necessary [S2, unofficial].
    *   *CityGaussian Specific:* Apply depth regularization and visibility-aware training to ensure consistency across block boundaries [S2, unofficial].
3.  **Merging:** At rendering time, combine the Gaussians from all blocks.
    *   For overlapping regions, apply feathered blending or weighted averaging to smooth discontinuities [S1, S2, unofficial].
    *   The merge operation is typically a simple union of Gaussian sets with a blending function $W(\mathbf{x})$ for points in overlap zones.
4.  **Level of Detail (LoD) / Streaming:** To support large-scale streaming, a hierarchical LoD pyramid can be constructed over the blocks or within blocks (e.g., using Hierarchical-GS or Scaffold-GS techniques) to allow progressive loading [S2, unofficial].

### Formulas
*   **Blending in Overlap:** For a point $\mathbf{x}$ in the overlap of blocks $A$ and $B$, the final color $C(\mathbf{x})$ is:
    $$ C(\mathbf{x}) = \frac{w_A(\mathbf{x}) C_A(\mathbf{x}) + w_B(\mathbf{x}) C_B(\mathbf{x})}{w_A(\mathbf{x}) + w_B(\mathbf{x})} $$
    where $w_A, w_B$ are spatially varying weights (e.g., distance-based falloff) [S1, S2, unofficial].
*   **Depth Regularization (CityGaussian):** Adds a term to the loss function to enforce geometric consistency between blocks:
    $$ \mathcal{L}_{total} = \mathcal{L}_{rgb} + \lambda_{depth} \mathcal{L}_{depth} + \lambda_{vis} \mathcal{L}_{visibility} $$
    where $\mathcal{L}_{depth}$ penalizes depth inconsistencies at block boundaries [S2, unofficial].

### Parameters
*   **Block Size:** Determined by GPU memory and scene density. Not given in the sources.
*   **Overlap Width:** Sufficient to cover the feathering region. Not given in the sources.
*   **Depth Regularization Weight ($\lambda_{depth}$):** Not given in the sources.
*   **Visibility Weight ($\lambda_{vis}$):** Not given in the sources.

### Edge Cases and Pitfalls
*   **Boundary Artifacts:** Discontinuities in color or geometry at block edges if blending is not handled correctly [S1, S2, unofficial].
*   **Lighting Variations:** Outdoor scenes have changing lighting; training blocks independently may cause inconsistent appearance [S2, unofficial].
*   **Occlusion Consistency:** Visibility-aware training is required to ensure that occlusions are consistent across block boundaries [S2, unofficial].

## Subtopics

### CityGS-X
A method that extends block-based partitioning with a geometry-aware architecture that refines block boundaries according to building footprints, achieving highly accurate urban reconstruction [S1, unofficial]. It targets large-scale urban scenes by combining semantic segmentation to allocate Gaussians only on building facades and road surfaces [S1, unofficial].

### Hierarchical 3D Gaussian Representations
A method that renders very large datasets in real time by storing a multi-resolution pyramid of Gaussians [S1, unofficial]. It is used in conjunction with block-based methods to enable streaming and efficient rendering of far-away details [S2, unofficial].

### Proxy-GS
A method that uses a coarse proxy mesh to guide Gaussian placement, significantly reducing the number of Gaussians required for background regions [S1, unofficial]. It can be integrated into the block training process to optimize storage for non-critical areas [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Training time scales linearly with the number of blocks and the number of images per block. Total training time is roughly the sum of training times for each block, allowing parallelization across GPUs/nodes [S1, S2, unofficial].
*   **Memory Complexity:** Memory usage is bounded by the largest block, enabling training on scenes that exceed single-GPU memory [S1, S2, unofficial].
*   **Measured Performance:** VastGaussian and CityGaussian enable rendering of city-scale scenes (square kilometers) [S2, unofficial]. DoGaussian supports distributed training across nodes [S2, unofficial].
*   **Strengths:** Scalability to very large scenes; parallelizable training; modular updates (blocks can be updated independently) [S1, S2, unofficial].
*   **Limitations:** Potential for artifacts at block boundaries; requires careful handling of lighting and visibility across blocks [S1, S2, unofficial].
*   **When to Use:** Large-scale urban reconstruction, autonomous driving simulation, city-scale digital twins [S1, S2, unofficial].
*   **When Not to Use:** Small-scale scenes where monolithic 3DGS is sufficient and simpler to implement [S1, unofficial].

## Background
The methods originated from the need to scale 3D Gaussian Splatting to city-scale environments. VastGaussian was presented at CVPR 2024, introducing block-based partitioning and decoupled appearance handling [S2, unofficial]. CityGaussian and CityGaussianV2 introduced hierarchical LoD and depth regularization for urban scenes [S2, unofficial]. DoGaussian extended this to distributed training across multiple nodes [S2, unofficial]. These methods build on the foundational 3DGS work [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
