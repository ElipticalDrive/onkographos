# 3D Gaussian Splatting LOD methods

Also called: RTGS, Hierarchical 3DGS, LODGE, EAGLES, SOGS, TRIPS, Scale-Adaptive Gaussian Splatting, Compact3D, Budgeted Gaussian Splatting, Iterative Gaussian Synopsis, Virtual Memory for 3D Gaussian Splatting, Gaussian clustering and merging, LightGaussian, Multi-scale filtering, CityGaussian, VastGaussian, DoGaussian, Octree-GS

## Overview
3D Gaussian Splatting (3DGS) Level-of-Detail (LOD) methods address the high memory footprint and rendering latency of monolithic Gaussian representations by reducing redundancy, compressing attributes, or organizing primitives into hierarchical structures [S1, S2, unofficial][S3]. Unlike standard 3DGS which renders all primitives, these methods enable efficient rendering of large-scale scenes or low-bandwidth deployment by dynamically selecting detail levels, pruning low-importance Gaussians, or quantizing attributes [S1, S2, unofficial][S3]. They set themselves apart from sibling methods like [Hierarchical Splatting](1168-hierarchical-splatting.md) by specifically targeting the unique attribute structure of 3D Gaussians (position, scale, rotation, opacity, and spherical harmonics) rather than generic point clouds or voxel grids [S2, unofficial][S3].

## How it works
The implementation of 3DGS LOD methods varies significantly by variant, but generally follows a pipeline of redundancy reduction, hierarchical construction, or attribute compression. The following sections detail the core mechanisms found in the primary variants.

### General Redundancy Reduction and Pruning
Methods like RTGS and GaussianSPA reduce the primitive count by removing redundant Gaussians [S1, unofficial].

1.  **Inputs:** A set of optimized 3D Gaussians $\mathcal{G} = \{g_i\}$, where each $g_i$ has position $\mu_i$, covariance $\Sigma_i$ (or scale $s_i$ and rotation $q_i$), opacity $\alpha_i$, and spherical harmonic coefficients $c_{i, l, m}$.
2.  **Redundancy Metric:** Calculate a redundancy score for each Gaussian based on spatial overlap and temporal consistency [S1, unofficial].
    *   **Spatial Overlap:** Measure the intersection of projected 2D Gaussians in screen space. If two Gaussians $g_i$ and $g_j$ have a high overlap ratio $O_{ij} > \tau_{overlap}$, one is a candidate for removal [S1, G, unofficial].
    *   **Temporal Consistency:** For dynamic scenes, compare the position/rotation of $g_i$ across frames. High variance indicates instability or redundancy [S1, G, unofficial].
3.  **Pruning:** Remove Gaussians with the lowest importance score (derived from loss gradients or projected area) that exceed the redundancy threshold [S1, unofficial][S3].
4.  **Outputs:** A sparse set of Gaussians $\mathcal{G}' \subset \mathcal{G}$.

### Hierarchical Construction (Octree/Cluster-based)
Methods like Hierarchical 3DGS and Octree-GS build a multi-resolution structure [S2, unofficial][S3].

1.  **Data Structure:** Construct an octree or cluster tree where each node represents a spatial region [S2, unofficial][S3].
    *   **Leaf Nodes:** Contain the original 3D Gaussians.
    *   **Internal Nodes:** Contain a "merged" or "representative" Gaussian that summarizes the radiance, opacity, and position of its children [S3].
2.  **Aggregation:** For an internal node $N$ with children $\mathcal{C}_N$, compute the aggregated Gaussian $g_N$:
    *   **Position:** Weighted mean of children positions: $\mu_N = \frac{\sum_{g \in \mathcal{C}_N} \alpha_g \mu_g}{\sum_{g \in \mathcal{C}_N} \alpha_g}$ [G].
    *   **Covariance:** Weighted sum of covariances plus the covariance of the means: $\Sigma_N = \frac{\sum_{g \in \mathcal{C}_N} \alpha_g (\Sigma_g + (\mu_g - \mu_N)(\mu_g - \mu_N)^T)}{\sum_{g \in \mathcal{C}_N} \alpha_g}$ [G].
    *   **Opacity:** Sum of children opacities (clamped to 1): $\alpha_N = \min(1, \sum_{g \in \mathcal{C}_N} \alpha_g)$ [G].
    *   **Color/SH:** Average or weighted average of children's SH coefficients [G].
3.  **LOD Selection:** During rendering, for each pixel or tile, traverse the tree from the root. Select the finest node $N$ such that the projected area of $g_N$ fits within the pixel grid or satisfies an error metric [S2, unofficial][S3].
4.  **Rendering:** Rasterize the selected Gaussians using standard alpha blending [S2, unofficial][S3].

### Attribute Compression and Quantization
Methods like EAGLES, SOGS, and Compact3D compress the Gaussian attributes [S3].

1.  **Quantization:**
    *   **Position/Scale/Rotation:** Quantize to 8–16 bits [S2, unofficial][S3].
    *   **Opacity:** Quantize to 8 bits (raw logit) [S2, unofficial][S3]. Note: EAGLES shows this reduces floaters [S3]. Compact3D avoids quantizing position/opacity to prevent overlapping artifacts [S3].
    *   **Spherical Harmonics (SH):** Distill to band 1 (DC color) and quantize higher bands to INT6 or use vector quantization [S2, unofficial][S3].
2.  **Grid-based Storage (SOGS):**
    *   Arrange Gaussian attributes into multiple 2D grids [S3].
    *   Sort grid values and apply smoothness regularization to penalize pixels with values different from their local neighborhood [S3].
    *   Compress RGB grid with JPEG XL; store other attributes as 32-bit OpenEXR images with zip compression [S3].
3.  **Neural Features (TRIPS):**
    *   Attach a neural feature vector to each Gaussian [S3].
    *   Render pyramid-like image feature planes according to projected Gaussian size to resolve blur [S3].
4.  **Outputs:** Compressed bitstream or grid files.

### Scale-Adaptive Filtering
Methods like Scale-Adaptive Gaussian Splatting (SA-GS) address aliasing [S3].

1.  **Inputs:** Rendered image, camera intrinsics, and Gaussian scales.
2.  **Filtering:** Apply an adaptive 2D low-pass filter at test time [S3].
    *   The filter cutoff frequency is adjusted based on the rendering resolution and camera distance [S3].
    *   This constrains the frequency content of the splats to prevent high-frequency noise (aliasing) when viewed from far distances or low resolutions [S3].
3.  **Outputs:** Anti-aliased image.

### Virtual Memory and Streaming
Methods like Vmem handle scenes larger than GPU memory [S1, unofficial].

1.  **Page Grouping:** Group Gaussians into fixed-size pages [S1, unofficial].
2.  **Proxy Mesh:** Construct a proxy mesh for visibility culling [S1, unofficial].
3.  **Streaming:** Stream only visible pages into a limited GPU buffer [S1, unofficial].
4.  **Outputs:** Rendered image with dynamic LOD based on memory availability.

## Subtopics

### [CityGaussian / VastGaussian / DoGaussian](6143-citygaussian-vastgaussian-dogaussian.md)
CityGaussian, VastGaussian, and DoGaussian are block-based partitioning methods that enable the training and rendering of city-scale scenes by dividing large environments into overlapping spatial tiles, training independent 3D Gaussian Splatting (3DGS) models per tile, and merging them at render time. They solve the memory and convergence limitations of monolithic 3DGS training on scenes spanning square kilometers. Unlike anchor-based hierarchical methods like Octree-GS or Scaffold-GS that build a single unified structure, these methods decouple training into independent blocks, allowing for distributed training and explicit handling of lighting variations across large areas.

### [Octree-GS](7341-octree-gs.md)
Octree-GS is a hierarchical level-of-detail (LOD) method for 3D Gaussian Splatting that builds an octree over the scene, associating each node with a merged Gaussian that summarizes the radiance and opacity of its children. It introduces an LOD strategy to better capture details, building upon the Scaffold-GS representation. It sets itself apart from sibling methods like CityGaussian by using a strict octree hierarchy for merging rather than voxel-grid scaffolding or clustering.

### [LightGaussian](8036-lightgaussian.md)
LightGaussian is a compression method for 3D Gaussian Splatting that reduces model size and improves rendering speed by combining structured pruning with Spherical Harmonic (SH) factorization and vector quantization. It achieves significant size reduction (up to 15x) and high frame rates (>200 FPS) with minimal quality degradation, distinguishing itself from sibling methods like CityGaussian by focusing on attribute-level compression (SH and position) rather than purely spatial clustering or octree-based spatial pruning.

### [Multi-scale filtering](8244-multi-scale-filtering.md)
Multi-scale filtering addresses spatial aliasing and performance degradation in 3D Gaussian Splatting (3DGS) when rendering at varying resolutions or camera distances. Unlike standard 3DGS, which suffers from dilation/erosion artifacts and high-frequency noise due to fixed Gaussian scales, multi-scale filtering constrains the representation's frequency content. It achieves this through 3D smoothing during optimization and resolution-aware selection or filtering during rendering, ensuring alias-free images across sampling rates.

### [Virtual Memory for 3D Gaussian Splatting](8403-virtual-memory-for-3d-gaussian-splatting.md)
Virtual Memory for 3D Gaussian Splatting (Vmem) enables the rendering of large-scale 3D Gaussian Splatting (3DGS) scenes that exceed available GPU memory by applying virtual texturing concepts. It groups Gaussians into fixed-size pages, uses a proxy mesh for visibility culling, and streams only visible pages into a limited GPU buffer. Unlike standard 3DGS which loads all Gaussians or relies on simple distance culling, Vmem separates virtual and physical address spaces, allowing dynamic LOD selection and just-in-time memory allocation to maintain stable frame rates in memory-constrained environments.

### [Gaussian clustering and merging](8436-gaussian-clustering-and-merging.md)
Gaussian clustering and merging is a family of level-of-detail (LOD) techniques for 3D Gaussian Splatting that reduces the number of splats by grouping them into representative clusters. Unlike [MIP-Splatting](6063-mip-splatting.md), which handles frequency aliasing, or [CityGaussian / VastGaussian / DoGaussian](6143-citygaussian-vastgaussian-dogaussian.md), which use spatial clustering for pruning, this family focuses on substitutive merging where the merged splat replaces the group entirely. It builds on [3D Gaussian Splatting LOD methods](5439-3d-gaussian-splatting-lod-methods.md).

### [Iterative Gaussian Synopsis](9289-iterative-gaussian-synopsis.md)
Iterative Gaussian Synopsis is a top-down strategy for constructing a hierarchy of Level of Detail (LOD) representations for 3D Gaussian Splatting (3DGS) [1]. It begins with a full-fidelity model and progressively derives coarser levels through adaptive, learnable pruning of anchors and downsampling of hierarchical grids, ensuring coherent simplification without the error accumulation inherent in bottom-up training. Unlike bottom-up approaches (e.g., [LapisGS](6143-lapisgs.md)) that build coarse layers first and add enhancements, this method derives coarser levels from the full-fidelity model, ensuring inter-layer consistency. It differs from independent per-resolution models (e.g., [Scaffold-GS](6143-scaffold-gs.md)) by sharing a foundational codebook and grids, extracting all LODs from a single optimization.

### [Budgeted Gaussian Splatting](9672-budgeted-gaussian-splatting.md)
Budgeted Gaussian Splatting (BCT) is a training and deployment method for large-scale 3D Gaussian Splatting that enables variable primitive counts (Level of Detail) without retraining, using a continuous budget-quality curve to select primitives. It sets itself apart from sibling methods like [CityGaussian / VastGaussian / DoGaussian](6143-citygaussian-vastgaussian-dogaussian.md) and [Octree-GS](7341-octree-gs.md) by decoupling the LOD selection from the training process, allowing a single trained model to be exported to any standard rasterizer with different budgets, achieving near-lossless quality at low compression ratios.

### Hierarchical 3DGS
Hierarchical 3DGS is a cluster-based Level of Detail (LoD) method developed by Inria, capable of rendering large scenes like Mont Saint-Michel interactively [S2, unofficial]. It uses an octree-like tree of clusters and renders only the relevant LoD, achieving a 99% reduction in primitive count at far distances [S2, unofficial].

### LODGE
LODGE is a level-of-detail framework for large-scale 3D Gaussian Splatting that enables efficient rendering by dynamically selecting appropriate detail levels [S1, unofficial]. It is part of Google's approach to handling large-scale scenes with hierarchical structures [S1, unofficial].

### EAGLES
EAGLES is a method that quantizes all Gaussian attributes including color, position, opacity, rotation, and scaling [S3]. A key pitfall noted is that quantization of opacity can lead to fewer floaters or visual artifacts in novel view synthesis tasks [S3]. On the MipNeRF360 dataset, EAGLES achieves SSIM 0.81, PSNR 27.15, LPIPS 0.24, and a size of 68 MB [S3].

### SOGS (Sparse Orthographic Grid Splatting)
SOGS arranges Gaussian attributes into multiple 2D grids, sorts them, and applies a smoothness regularization to penalize pixels with values very different from their local neighborhood on the 2D grid [S3]. For storage, SOGS compresses the RGB grid with JPEG XL and stores all other attributes as 32-bit OpenEXR images with zip compression [S3]. On the MipNeRF360 dataset, SOGS achieves SSIM 0.801, PSNR 26.98, LPIPS 0.238, and a size of 28.80 MB [S3].

### 3D Gaussian Splatting Compression
This compression method does not quantize opacity and position because sharing them results in overlapping Gaussians [S3]. On the MipNeRF360 dataset, Compact3D achieves SSIM 0.808, PSNR 27.16, LPIPS 0.228, and a size of / MB (unspecified) [S3].

### TRIPS
TRIPS attaches a neural feature onto Gaussians [S3]. It renders pyramid-like image feature planes according to the projected Gaussians' size, similar to ADOP, to resolve blur issues [S3].

### Scale-Adaptive Gaussian Splatting
Scale-Adaptive Gaussian Splatting utilizes an adaptive 2D low-pass filter at test time, adjusted according to the rendering resolution and camera distance [S3].

## Operating characteristics
*   **Memory Reduction:** Methods like Hierarchical 3DGS achieve up to 99% reduction in primitive count at far distances [S2, unofficial]. LightGaussian achieves up to 15x size reduction [S2, unofficial]. SOGS achieves a size of 28.80 MB on MipNeRF360 [S3].
*   **Rendering Speed:** LightGaussian achieves >200 FPS [S1, unofficial]. Hierarchical methods reduce rendering load by culling distant/coarse primitives [S2, unofficial].
*   **Quality Trade-offs:** Quantization (EAGLES, SOGS) may introduce artifacts if not carefully tuned (e.g., opacity quantization) [S3]. Pruning (RTGS, LightGaussian) may lose fine details if the importance metric is inaccurate [S1, unofficial].
*   **Scalability:** Block-based methods (CityGaussian) and Virtual Memory (Vmem) enable rendering of city-scale scenes (square kilometers) [S1, S2, unofficial].
*   **Aliasing:** Multi-scale filtering and SA-GS reduce aliasing artifacts at varying resolutions [S1, unofficial][S3].

## Background
3D Gaussian Splatting was introduced as a real-time radiance field representation [S1, unofficial][S3]. As the technique gained popularity, researchers identified limitations in memory usage and rendering performance for large scenes. This led to the development of various LOD and compression methods, including hierarchical structures (Octree-GS, Hierarchical 3DGS), compression techniques (LightGaussian, EAGLES, SOGS), and large-scale training strategies (CityGaussian, VastGaussian) [S1, S2, unofficial][S3]. These methods build upon the foundational 3DGS rasterization pipeline [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
