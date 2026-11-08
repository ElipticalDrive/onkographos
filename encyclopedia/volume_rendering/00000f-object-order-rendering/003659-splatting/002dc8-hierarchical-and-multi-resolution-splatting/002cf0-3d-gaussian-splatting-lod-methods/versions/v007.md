# 3D Gaussian Splatting LOD methods

Also called: RTGS, Hierarchical 3DGS, LODGE, EAGLES, SOGS, Compact3D, TRIPS

## Overview
3D Gaussian Splatting (3DGS) Level-of-Detail (LOD) and compression methods address the excessive memory footprint and rendering latency of monolithic 3D Gaussian representations by reducing redundancy and adapting resolution to viewing distance [S1, unofficial][S3]. Unlike standard 3DGS which renders all primitives, these methods employ hierarchical structures, clustering, or attribute quantization to select a subset of Gaussians or coarser approximations for a given pixel or region [S1, unofficial][S3]. They set themselves apart from sibling methods like [Hierarchical Splatting](1168-hierarchical-splatting.md) by specifically optimizing the Gaussian parameter space (position, covariance, spherical harmonics) for spatial overlap and frequency content, rather than relying solely on voxel or point cloud hierarchies [S1, unofficial][S3].

## How it works
The methods vary significantly in their approach to LOD, ranging from spatial clustering to attribute quantization. The following sections detail the core mechanisms for the major variants identified in the sources.

### General Redundancy Reduction (RTGS)
RTGS reduces the number of Gaussians by removing those with high spatial overlap and low temporal consistency [S1, unofficial].

1.  **Input:** A set of 3D Gaussians $\mathcal{G} = \{g_i\}$ with positions $\mu_i$, covariances $\Sigma_i$, opacities $\alpha_i$, and colors $c_i$.
2.  **Spatial Overlap Calculation:** Compute the pairwise spatial overlap or influence between Gaussians. Gaussians contributing to the same pixel region with high redundancy are identified [S1, unofficial].
3.  **Temporal Consistency:** For dynamic scenes or multi-view consistency, evaluate the stability of Gaussian attributes over time or across views [S1, unofficial].
4.  **Pruning:** Remove Gaussians that exceed a redundancy threshold based on the combined spatial and temporal metrics [S1, unofficial].
5.  **Output:** A sparse set of Gaussians $\mathcal{G}' \subset \mathcal{G}$ [S1, unofficial].

### Hierarchical 3DGS (Inria)
This method uses an octree-like tree of clusters to enable LOD rendering [S2, unofficial].

1.  **Clustering:** Group Gaussians into clusters based on spatial proximity.
2.  **Hierarchy Construction:** Organize clusters into an octree structure. Each node in the octree represents a merged Gaussian or a set of child clusters [S2, unofficial].
3.  **LOD Selection:** During rendering, traverse the octree. If a node's projected footprint is small enough (or error threshold met), render the merged Gaussian at that node. Otherwise, traverse to children [S2, unofficial].
4.  **Merging:** Internal nodes store a "merged" Gaussian with weighted mean position, covariance, and opacity, summarizing the radiance and opacity of its children [S2, unofficial][Parent](5618-hierarchical-and-multi-resolution-splatting.md).
5.  **Performance:** Achieves up to 99% reduction in primitive count at far distances [S2, unofficial].

### LODGE
LODGE is a framework for large-scale scenes that dynamically selects detail levels [S1, unofficial].

1.  **Scene Decomposition:** Decompose the large-scale scene into spatial regions or tiles.
2.  **LOD Pyramid:** Construct a multi-resolution representation for each region, similar to a Gaussian pyramid [S1, unofficial].
3.  **Dynamic Selection:** At render time, for each pixel or screen-space region, determine the appropriate LOD level based on camera distance and screen coverage [S1, unofficial].
4.  **Rendering:** Rasterize only the primitives from the selected LOD level [S1, unofficial].

### EAGLES
EAGLES focuses on quantizing all Gaussian attributes to reduce size [S3].

1.  **Quantization:** Quantize position, scaling, rotation, opacity, and color (SH coefficients) to lower bit depths [S3].
    *   Position and scaling: 8–16 bits [S2, unofficial].
    *   Opacity: 8 bits (raw logit) [S2, unofficial].
2.  **Entropy Coding:** Apply entropy coding (Huffman, range, or zstd) to the quantized attributes [S2, unofficial].
3.  **Artifact Mitigation:** Quantization of opacity is specifically tuned to reduce floaters and visual artifacts in novel view synthesis [S3].
4.  **Performance:** On MipNeRF360, achieves SSIM 0.81, PSNR 27.15, LPIPS 0.24, and size 68 MB [S3].

### SOGS (Sparse Orthographic Grid Splatting)
SOGS arranges attributes into 2D grids and applies smoothness regularization [S3].

1.  **Grid Mapping:** Map Gaussian attributes to multiple 2D grids.
2.  **Sorting:** Sort the grids to group similar attributes spatially [S3].
3.  **Regularization:** Apply a smoothness regularization term that penalizes pixels with values significantly different from their local neighborhood in the 2D grid [S3].
4.  **Compression:**
    *   RGB grid: Compressed with JPEG XL [S3].
    *   Other attributes: Stored as 32-bit OpenEXR images with zip compression [S3].
5.  **Performance:** On MipNeRF360, achieves SSIM 0.801, PSNR 26.98, LPIPS 0.238, and size 28.80 MB [S3].

### Compact3D
Compact3D avoids quantizing certain attributes to prevent overlap artifacts [S3].

1.  **Selective Quantization:** Do not quantize opacity and position, as sharing these values leads to overlapping Gaussians and visual artifacts [S3].
2.  **Compression:** Compress other attributes (SH, scaling, rotation) using standard techniques [S3].
3.  **Performance:** On MipNeRF360, achieves SSIM 0.808, PSNR 27.16, LPIPS 0.228 [S3]. Size is not specified in the sources [S3].

### TRIPS
TRIPS enhances rendering quality by attaching neural features to Gaussians [S3].

1.  **Feature Attachment:** Attach a neural feature vector to each Gaussian [S3].
2.  **Pyramid Rendering:** Render pyramid-like image feature planes according to the projected size of the Gaussians, similar to ADOP [S3].
3.  **Blur Resolution:** This multi-scale feature rendering resolves blur issues inherent in standard 3DGS splatting [S3].

## Subtopics

### [CityGaussian / VastGaussian / DoGaussian](6143-citygaussian-vastgaussian-dogaussian.md)
These methods partition large scenes into overlapping spatial tiles, training independent 3DGS models per tile and merging them at render time. They solve memory and convergence limitations for city-scale scenes by decoupling training, allowing distributed training and handling lighting variations. Unlike anchor-based methods, they use block-based partitioning. See [CityGaussian / VastGaussian / DoGaussian](6143-citygaussian-vastgaussian-dogaussian.md).

### [Octree-GS](7341-octree-gs.md)
Octree-GS builds a strict octree hierarchy over the scene, associating each node with a merged Gaussian summarizing children's radiance and opacity. It sets itself apart from CityGaussian by using a unified octree structure rather than independent tiles. See [Octree-GS](7341-octree-gs.md).

### [LightGaussian](8036-lightgaussian.md)
LightGaussian reduces model size and improves speed by combining structured pruning with Spherical Harmonic (SH) factorization and vector quantization. It focuses on attribute-level compression rather than spatial clustering. See [LightGaussian](8036-lightgaussian.md).

### [Scale-Adaptive Gaussian Splatting](8085-scale-adaptive-gaussian-splatting.md)
SA-GS addresses aliasing artifacts by employing an adaptive 2D low-pass filter during rendering, dynamically adjusted based on resolution and camera distance. It prevents high-frequency artifacts without changing the underlying Gaussian representation. See [Scale-Adaptive Gaussian Splatting](8085-scale-adaptive-gaussian-splatting.md).

### [Multi-scale filtering](8244-multi-scale-filtering.md)
Multi-scale filtering constrains the representation's frequency content through 3D smoothing during optimization and resolution-aware filtering during rendering, ensuring alias-free images across sampling rates. See [Multi-scale filtering](8244-multi-scale-filtering.md).

### [Virtual Memory for 3D Gaussian Splatting](8403-virtual-memory-for-3d-gaussian-splatting.md)
Vmem enables rendering of large scenes exceeding GPU memory by grouping Gaussians into pages, using a proxy mesh for culling, and streaming visible pages. It separates virtual and physical address spaces for dynamic LOD selection. See [Virtual Memory for 3D Gaussian Splatting](8403-virtual-memory-for-3d-gaussian-splatting.md).

### [Gaussian clustering and merging](8436-gaussian-clustering-and-merging.md)
This family reduces primitive counts by substituting groups of Gaussians with representative merged splats. It differs from MIP-Splatting (frequency aliasing) and CityGaussian (spatial pruning for streaming) by focusing on substitutive merging. See [Gaussian clustering and merging](8436-gaussian-clustering-and-merging.md).

### [Iterative Gaussian Synopsis](9289-iterative-gaussian-synopsis.md)
This top-down strategy constructs LOD hierarchies by progressively pruning anchors and downsampling grids from a full-fidelity model, ensuring inter-layer consistency. It differs from bottom-up approaches like LapisGS and independent models like Scaffold-GS by sharing a foundational codebook. See [Iterative Gaussian Synopsis](9289-iterative-gaussian-synopsis.md).

### [Budgeted Gaussian Splatting](9672-budgeted-gaussian-splatting.md)
BCT enables variable primitive counts (LOD) without retraining using a continuous budget-quality curve. It decouples LOD selection from training, allowing a single model to be exported with different budgets. See [Budgeted Gaussian Splatting](9672-budgeted-gaussian-splatting.md).

## Operating characteristics
*   **Memory Reduction:** Methods like Hierarchical 3DGS achieve up to 99% reduction in primitive count at far distances [S2, unofficial]. EAGLES reduces size to 68 MB on MipNeRF360 [S3]. SOGS achieves 28.80 MB [S3].
*   **Rendering Speed:** LightGaussian achieves over 200 FPS with slight quality loss [S1, unofficial]. LODGE and Hierarchical 3DGS enable interactive rendering of large scenes like Mont Saint-Michel [S2, unofficial].
*   **Quality Trade-offs:** Quantization (EAGLES) and pruning (LightGaussian) introduce minor quality degradation, measured by SSIM, PSNR, and LPIPS [S3]. Compact3D avoids position/opacity quantization to maintain higher fidelity [S3].
*   **Scalability:** Block-based methods (CityGaussian) and Virtual Memory (Vmem) enable city-scale rendering [S1, S2, unofficial].
*   **Aliasing:** Scale-Adaptive and Multi-scale filtering methods specifically target aliasing artifacts at varying resolutions [S1, unofficial].

## Background
3D Gaussian Splatting was introduced as a real-time radiance field representation [S1, unofficial]. As applications expanded to large-scale urban scenes and high-resolution displays, the memory and computational demands of storing and rendering millions of Gaussians became a bottleneck [S1, S2, unofficial]. This led to the development of LOD and compression methods, including hierarchical structures (Octree-GS, Hierarchical 3DGS), clustering (CityGaussian), and attribute quantization (EAGLES, LightGaussian) [S1, S2, unofficial][S3]. These methods build upon the foundational 3DGS rasterization pipeline [Parent](23-splatting.md).

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
