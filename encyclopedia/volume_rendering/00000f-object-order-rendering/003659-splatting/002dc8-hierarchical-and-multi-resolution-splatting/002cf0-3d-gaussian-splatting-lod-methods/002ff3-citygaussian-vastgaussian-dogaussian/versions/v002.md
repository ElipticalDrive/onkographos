# CityGaussian / VastGaussian / DoGaussian

Also called: City-Gaussian, CityGaussianV2, DoGaussian

## Overview
CityGaussian, VastGaussian, and DoGaussian are methods for rendering large-scale urban environments by partitioning the scene into spatial blocks or tiles, training independent 3D Gaussian Splatting (3DGS) models for each, and merging them at render time [S1, S2, unofficial]. They solve the scalability problem of training and storing monolithic 3DGS models for square-kilometer scenes by enabling distributed training and memory-efficient streaming via Level of Detail (LoD) pyramids [S2, unofficial]. Unlike sibling methods that use hierarchical data structures (like Octree-GS) or clustering (like Iterative Gaussian Synopsis) to reduce Gaussian count, these methods rely on spatial decomposition and overlap merging to handle lighting variations and boundary artifacts [S1, S2, unofficial].

## How it works
**Inputs:** A set of calibrated images covering a large urban area; camera poses.
**Outputs:** A set of per-block 3DGS models (positions, covariances, colors/SH coefficients, opacities) and a merge/LoD structure for rendering.

**Data Structures:**
1.  **Block Grid:** The scene is divided into a grid of spatially disjoint or overlapping tiles/blocks.
    *   *VastGaussian:* Spatially disjoint regions [S2, unofficial].
    *   *CityGaussian/DoGaussian:* Overlapping blocks to handle boundaries [S1, S2, unofficial].
2.  **Per-Block Model:** Standard 3DGS representation for each block:
    *   Positions $\mathbf{p} \in \mathbb{R}^3$
    *   Scales $\mathbf{s} \in \mathbb{R}^3$
    *   Rotations $\mathbf{q} \in \mathbb{R}^4$ (quaternion)
    *   Opacity $\alpha \in \mathbb{R}$
    *   Spherical Harmonics coefficients $\mathbf{c}_{lm}$
3.  **LoD Pyramid (CityGaussian):** A hierarchical structure where Gaussians are clustered or pruned at different resolutions to support streaming [S2, unofficial].

**Steps:**
1.  **Partitioning:** Divide the bounding box of the scene into blocks.
    *   *VastGaussian:* Use spatially disjoint tiles [S2, unofficial].
    *   *CityGaussian:* Use overlapping tiles [S1, S2, unofficial].
2.  **Independent Training:** Train a standard 3DGS model for each block independently.
    *   *VastGaussian:* Apply decoupled appearance handling to manage varying lighting conditions across blocks [S2, unofficial].
    *   *CityGaussian:* Use depth regularization and visibility-aware training to improve consistency [S2, unofficial].
    *   *DoGaussian:* Distribute the training of different blocks across multiple GPU nodes [S2, unofficial].
3.  **Merging:** Combine the per-block models for rendering.
    *   *VastGaussian:* Merge with feathered overlap to smooth boundaries [S2, unofficial].
    *   *CityGaussian:* Use the LoD pyramid to stream relevant Gaussians based on view distance [S2, unofficial].
4.  **Rendering:** Rasterize the merged Gaussians. For CityGaussian, select the appropriate LoD level based on the camera's distance to the block [S2, unofficial].

**Parameters:**
*   Block size: Not given in the sources.
*   Overlap size: Not given in the sources.
*   LoD levels: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Boundary Artifacts:** Disjoint blocks (VastGaussian) require careful merging or feathering to avoid visible seams [S2, unofficial]. Overlapping blocks (CityGaussian) require blending strategies [S1, S2, unofficial].
*   **Lighting Inconsistency:** Independent training can lead to discontinuous lighting across block boundaries. VastGaussian addresses this with decoupled appearance handling [S2, unofficial].
*   **Training Consistency:** Distributed training (DoGaussian) may suffer from lack of global context. CityGaussian uses depth regularization to mitigate this [S2, unofficial].

## Subtopics

### CityGS-X
A method that extends block-based partitioning with a geometry-aware architecture that refines block boundaries according to building footprints, achieving highly accurate urban reconstruction [S1, unofficial]. It targets large-scale urban scenes by combining semantic segmentation to allocate Gaussians only on building facades and road surfaces [S1, unofficial].

### Hierarchical 3D Gaussian Representations
A method that renders very large datasets in real time by storing a multi-resolution pyramid of Gaussians [S1, unofficial].

### Proxy-GS
A method that uses a coarse proxy mesh to guide Gaussian placement, significantly reducing the number of Gaussians required for background regions [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Training time is linear with the number of blocks, as each block is trained independently. This allows for parallelization across multiple GPUs (DoGaussian) [S2, unofficial].
*   **Memory Complexity:** Memory usage is proportional to the number of Gaussians per block. The total memory is the sum of all block models, but only relevant blocks need to be loaded into VRAM for rendering, enabling large scene support [S2, unofficial].
*   **Measured Performance:** CityGaussian supports rendering across square kilometers [S2, unofficial]. VastGaussian handles varying lighting conditions effectively [S2, unofficial].
*   **Strengths:** Scalability to city-scale scenes; distributed training capability; handling of dynamic lighting (VastGaussian); streaming support (CityGaussian).
*   **Limitations:** Potential for boundary artifacts if merging is not handled well; independent training may lack global semantic consistency compared to monolithic models.
*   **When to use:** Large-scale urban reconstruction, autonomous driving simulation, scenes requiring distributed training resources.
*   **When not to use:** Small-scale scenes where monolithic 3DGS is sufficient; scenes with highly dynamic global lighting changes that are hard to decouple per block.

## Background
CityGaussian, VastGaussian, and DoGaussian were developed to address the limitations of 3DGS in large-scale urban environments. VastGaussian was presented at CVPR 2024 [S2, unofficial]. CityGaussian and its variants (CityGaussianV2, CityGS-X) focus on hierarchical LoD and geometry-aware partitioning [S1, S2, unofficial]. DoGaussian focuses on distributed training [S2, unofficial]. These methods build on the foundational 3DGS work [S1, unofficial] and are part of the broader effort to scale neural rendering to city-scale datasets like MatrixCity [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
