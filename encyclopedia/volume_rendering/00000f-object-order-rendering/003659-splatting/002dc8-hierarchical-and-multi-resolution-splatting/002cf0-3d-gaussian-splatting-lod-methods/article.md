# 3D Gaussian Splatting LOD methods

Also called: RTGS

## Overview
3D Gaussian Splatting Level-of-Detail (LOD) methods address the computational and memory bottlenecks of rendering large-scale scenes with millions of 3D Gaussians by reducing redundancy and selecting appropriate detail levels based on viewing distance or screen footprint [S1, S2, unofficial][S3]. Unlike standard 3D Gaussian Splatting which renders all primitives, these methods employ hierarchical structures, clustering, or compression to achieve significant size reductions (up to 99% at far distances) while maintaining interactive frame rates [S2, unofficial][S3]. They set themselves apart from sibling methods like [Hierarchical Splatting](1168-hierarchical-splatting.md) by specifically adapting the Gaussian primitive representation (merging, pruning, or quantizing attributes) rather than just organizing voxels or points, and from [Multi-Resolution Points](4784-multi-resolution-points.md) by handling the anisotropic, view-dependent nature of Gaussian splats [S1, S2, unofficial].

## How it works
The general approach involves constructing a multi-resolution representation of the Gaussian scene and dynamically selecting the appropriate level during rendering. The specific implementation varies by variant (see Subtopics), but the core logic follows these steps:

1.  **Hierarchy Construction**: The initial set of $N$ Gaussians is organized into a hierarchy. Common strategies include:
    *   **Octree/Cluster Aggregation**: Gaussians are grouped into clusters or octree nodes. Each internal node stores a "merged" Gaussian, typically calculated as a weighted average of its children's mean positions, covariances, opacities, and spherical harmonic (SH) coefficients [S1, S2, unofficial].
    *   **Grid/Anchor Association**: Gaussians are associated with a sparse grid or anchor points, where attributes are interpolated or quantized based on proximity [S3].
2.  **Selection Metric**: For each pixel or screen-space region, the renderer determines which level of the hierarchy to use.
    *   **Screen-Space Footprint**: The projected size of the Gaussian (or cluster) is compared to the pixel size. If the footprint is small, a coarser LOD (larger, merged Gaussian) is used [S3].
    *   **Distance/Depth**: For large scenes, distance from the camera determines the LOD, with far-away regions using highly aggregated representations [S2, unofficial].
    *   **Redundancy Reduction**: Spatial overlap and temporal consistency are analyzed to prune redundant Gaussians [S1, unofficial].
3.  **Rendering**: The selected primitives are rasterized.
    *   **Merged Gaussian Splatting**: The merged Gaussian from a cluster node is splatted directly. The covariance of the merged Gaussian must be computed to ensure it covers the spatial extent of the children without excessive overlap artifacts [S1, unofficial].
    *   **Compression/Quantization**: If using methods like [EAGLES](8036-eagles.md) or [SOGS](8036-sogs.md), attributes are decoded from compressed grids or codebooks before splatting [S3].

**Data Structures**:
*   **Octree Node**: Contains a merged Gaussian (mean $\mu$, covariance $\Sigma$, opacity $\alpha$, SH coefficients $c$) and a pointer to children or a leaf list of original Gaussians [S1, unofficial].
*   **Cluster**: Similar to octree node, but grouping may be based on spatial proximity or importance scores rather than strict geometric subdivision [S2, unofficial].

**Formulas**:
*   **Merged Gaussian Mean**: $\mu_{merged} = \frac{\sum w_i \mu_i}{\sum w_i}$, where $w_i$ is the importance or opacity weight of child $i$ [S1, unofficial].
*   **Merged Gaussian Covariance**: $\Sigma_{merged} = \frac{\sum w_i (\Sigma_i + \mu_i \mu_i^T)}{\sum w_i} - \mu_{merged} \mu_{merged}^T$ [S1, unofficial]. This ensures the merged Gaussian covers the variance of its children.

**Parameters**:
*   **LOD Threshold**: A screen-space size threshold (e.g., in pixels) below which a cluster is rendered as a single primitive [S3].
*   **Quantization Bits**: Number of bits for position (e.g., 16-bit), scale/rotation (e.g., 8-16 bits), and opacity (e.g., 8-bit) [S2, unofficial][S3].

**Edge Cases**:
*   **Quantization Artifacts**: Quantizing opacity can lead to visual artifacts or "floaters" if not handled carefully [S3].
*   **Overlapping Gaussians**: Sharing quantized positions can cause overlapping Gaussians, degrading quality [S3].
*   **Blur**: Coarse LODs may introduce blur if the merged Gaussian does not accurately represent the high-frequency details of the children [S3].

## Subtopics

### [CityGaussian / VastGaussian / DoGaussian](6143-citygaussian-vastgaussian-dogaussian.md)
These methods enable city-scale rendering by partitioning the scene into overlapping spatial tiles, training independent 3DGS models per tile, and merging them at render time. They solve memory and convergence limitations of monolithic training on square-kilometer scenes. Unlike anchor-based hierarchical methods, they decouple training into independent blocks, allowing distributed training and handling lighting variations.

### [Octree-GS](7341-octree-gs.md)
Octree-GS builds a strict octree hierarchy over the scene, associating each node with a merged Gaussian that summarizes radiance and opacity of its children. It introduces an LOD strategy to capture details, building on Scaffold-GS. It differs from CityGaussian by using a strict octree for merging rather than voxel-grid scaffolding.

### [LightGaussian](8036-lightgaussian.md)
LightGaussian is a compression method that reduces model size and improves rendering speed by combining structured pruning with Spherical Harmonic (SH) factorization and vector quantization. It achieves up to 15x size reduction and >200 FPS with minimal quality degradation, focusing on attribute-level compression rather than purely spatial clustering.

### [Multi-scale filtering](8244-multi-scale-filtering.md)
Multi-scale filtering addresses spatial aliasing and performance degradation in 3DGS at varying resolutions or camera distances. It constrains the representation's frequency content through 3D smoothing during optimization and resolution-aware selection during rendering, ensuring alias-free images.

### [Virtual Memory for 3D Gaussian Splatting](8403-virtual-memory-for-3d-gaussian-splatting.md)
Virtual Memory for 3D Gaussian Splatting (Vmem) enables rendering of large-scale scenes exceeding GPU memory by applying virtual texturing concepts. It groups Gaussians into fixed-size pages, uses a proxy mesh for visibility culling, and streams only visible pages into a limited GPU buffer. It separates virtual and physical address spaces, allowing dynamic LOD selection and just-in-time memory allocation.

### [Gaussian clustering and merging](8436-gaussian-clustering-and-merging.md)
This family of LOD techniques reduces the number of splats by grouping them into representative clusters. Unlike MIP-Splatting, which handles frequency aliasing, or CityGaussian, which uses spatial clustering for pruning, this family focuses on substitutive merging where the merged splat replaces the group entirely.

### [Iterative Gaussian Synopsis](9289-iterative-gaussian-synopsis.md)
Iterative Gaussian Synopsis is a top-down strategy for constructing a hierarchy of LOD representations for 3DGS [1]. It begins with a full-fidelity model and progressively derives coarser levels through adaptive, learnable pruning of anchors and downsampling of hierarchical grids, ensuring coherent simplification without error accumulation inherent in bottom-up training. It differs from independent per-resolution models by sharing a foundational codebook and grids, extracting all LODs from a single optimization.

### [Budgeted Gaussian Splatting](9672-budgeted-gaussian-splatting.md)
Budgeted Gaussian Splatting (BCT) is a training and deployment method for large-scale 3DGS that enables variable primitive counts (Level of Detail) without retraining, using a continuous budget-quality curve to select primitives. It decouples LOD selection from the training process, allowing a single trained model to be exported to any standard rasterizer with different budgets, achieving near-lossless quality at low compression ratios.

### Hierarchical 3DGS
Hierarchical 3DGS uses an octree-like tree of clusters; renders only the relevant LoD; achieves 99% reduction at far distances [S2, unofficial]. It is a cluster-based Level of Detail (LoD) method developed by Inria, capable of rendering large scenes like Mont Saint-Michel interactively [S2, unofficial].

### LODGE
LODGE is a level-of-detail framework for large-scale 3D Gaussian Splatting that enables efficient rendering by dynamically selecting appropriate detail levels [S1, unofficial].

### EAGLES
EAGLES is a method that quantizes all Gaussian attributes including color, position, opacity, rotation, and scaling [S3]. Quantization of opacity leads to fewer floaters or visual artifacts in the novel view synthesis task [S3]. On the MipNeRF360 dataset, EAGLES achieves SSIM 0.81, PSNR 27.15, LPIPS 0.24, and a size of 68 MB [S3].

### SOGS (Sparse Orthographic Grid Splatting)
SOGS arranges Gaussian attributes into multiple 2D grids, sorts them, and applies a smoothness regularization to penalize pixels with values very different from their local neighborhood on the 2D grid [S3]. For storage, SOGS compresses the RGB grid with JPEG XL and stores all other attributes as 32-bit OpenEXR images with zip compression [S3]. On the MipNeRF360 dataset, SOGS achieves SSIM 0.801, PSNR 26.98, LPIPS 0.238, and a size of 28.80 MB [S3].

### 3D Gaussian Splatting Compression
This compression method does not quantize opacity and position because sharing them results in overlapping Gaussians [S3]. On the MipNeRF360 dataset, Compact3D achieves SSIM 0.808, PSNR 27.16, LPIPS 0.228, and a size of / MB (unspecified) [S3].

### TRIPS
TRIPS attaches a neural feature onto Gaussians [S3]. It renders pyramid-like image feature planes according to the projected Gaussians' size, similar to ADOP, to resolve blur issues [S3].

### Scale-Adaptive Gaussian Splatting
Scale-Adaptive Gaussian Splatting utilizes an adaptive 2D low-pass filter at test time, adjusted according to the rendering resolution and camera distance [S3].

## Operating characteristics
*   **Time Complexity**: Rendering time is reduced by a factor proportional to the LOD reduction ratio (e.g., 99% reduction at far distances [S2, unofficial]). However, hierarchy traversal or grid lookup adds overhead [S1, unofficial][S3].
*   **Memory Complexity**: Significant reduction in memory footprint, from hundreds of MBs to tens of MBs or less, depending on the compression method [S2, unofficial][S3].
*   **Measured Performance**: EAGLES achieves 68 MB size on MipNeRF360 [S3]. SOGS achieves 28.80 MB [S3]. LightGaussian achieves >200 FPS [S1, unofficial].
*   **Scaling**: Methods like CityGaussian and VastGaussian scale to city-sized scenes (square kilometers) [S2, unofficial]. Vmem scales to scenes exceeding GPU memory [S1, unofficial].
*   **Strengths**: Enables interactive rendering of large scenes, reduces storage requirements, mitigates aliasing [S1, S2, unofficial][S3].
*   **Limitations**: Potential for visual artifacts (blur, floaters) if LOD selection or quantization is not tuned correctly [S3]. Complexity in hierarchy construction and traversal [S1, unofficial].

## Background
3D Gaussian Splatting (3DGS) was introduced by Kerbl et al. in 2023 [S1, unofficial][S3]. The need for LOD methods arose due to the high computational and memory costs of rendering large numbers of Gaussians, especially in large-scale scenes [S1, S2, unofficial]. RTGS (Redundancy Reduction) was proposed to remove Gaussians based on spatial overlap and temporal consistency [S1, unofficial]. Hierarchical 3DGS was developed by Inria for large-scale interactive rendering [S2, unofficial]. LODGE was introduced by Google for large-scale scenes [S1, unofficial]. EAGLES, SOGS, and Compact3D are compression methods proposed to reduce model size [S3].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
