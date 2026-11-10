# 3D Gaussian Splatting LOD methods

Also called: RTGS, Hierarchical 3DGS, LODGE, EAGLES, SOGS, H-3DGS, SPTs, Virtual Memory for 3D Gaussian Splatting, Budgeted Gaussian Splatting, Iterative Gaussian Synopsis, Scale-Adaptive Gaussian Splatting, Multi-scale filtering, LightGaussian, Gaussian clustering and merging, Unreachable Gaussian Handling

## Overview
3D Gaussian Splatting (3DGS) Level-of-Detail (LOD) methods address the prohibitive memory consumption and rendering latency of monolithic 3D Gaussian representations, particularly in large-scale or high-fidelity scenes. While standard 3DGS requires storing and rasterizing all primitives, LOD methods decompose the scene into hierarchical structures or compressed representations to dynamically select the appropriate granularity based on camera distance, resolution, or hardware constraints. These methods distinguish themselves from sibling techniques like [Hierarchical Splatting](1168-hierarchical-splatting.md) by specifically adapting the 3DGS primitive (Gaussian) properties—such as covariance, opacity, and spherical harmonics—into multi-resolution hierarchies or quantized codebooks, rather than relying solely on generic point-cloud or voxel-based aggregation.

## How it works
The implementation of 3DGS LOD methods varies significantly by variant, but they generally follow a pattern of spatial partitioning, attribute compression, or hierarchical merging. Below are the core mechanisms for the primary families.

### Hierarchical and Tree-Based LOD (H-3DGS, Octree-GS, SPTs)
These methods build a spatial hierarchy (octree or sequential point tree) to manage LOD.

**Inputs:** A set of optimized 3D Gaussians $\mathcal{G} = \{g_i\}$, where each $g_i$ has position $\mu_i$, covariance $\Sigma_i$ (or scale $s_i$ and rotation $q_i$), opacity $\alpha_i$, and spherical harmonics coefficients $c_{i, l, m}$.
**Outputs:** A hierarchical structure (e.g., Octree or SPT) and a rendering pipeline that traverses this structure.

**Data Structures:**
1.  **Octree Node:** Stores aggregated Gaussian properties (mean position, merged covariance, total opacity) and a pointer to children or a leaf Gaussian.
2.  **Sequential Point Tree (SPT):** A dynamic tree structure where nodes represent clusters of Gaussians. Each node maintains a "cut" level indicating the current LOD.

**Algorithm: Hierarchical Construction and Rendering (Generic)**
1.  **Initialization:** Start with the full set of Gaussians $\mathcal{G}$.
2.  **Clustering/Merging:**
    *   Group Gaussians into spatial clusters (e.g., using an octree grid or anchor points).
    *   For each cluster, compute a representative Gaussian $g_{parent}$ by merging children properties.
    *   *Merging Formula (Approximate):* The mean $\mu_{parent}$ is the weighted average of children means. The covariance $\Sigma_{parent}$ is the weighted sum of children covariances plus the weighted outer product of mean differences. Opacity $\alpha_{parent}$ is the sum of children opacities (capped at 1.0). [S2, unofficial][S4, unofficial]
3.  **Hierarchy Construction:**
    *   Recursively subdivide clusters until a minimum size or maximum depth is reached.
    *   In SPTs, this is done dynamically during training via densification and pruning. [S4, unofficial]
4.  **Rendering Traversal:**
    *   For each pixel or tile, traverse the tree from the root.
    *   **Criterion:** Stop at node $N$ if the projected screen-space footprint of $N$ is smaller than a threshold $\tau$ (e.g., 1-2 pixels) OR if the error between $N$'s aggregate and its children is below a precision bound. [S3]
    *   **Render:** Rasterize the representative Gaussian of node $N$.
    *   **Fallback:** If the criterion is not met, recurse to children.

**Pitfalls:**
*   **Unreachable Gaussians:** In SPTs, a child Gaussian may become larger than its parent during training, violating the cut condition. These Gaussians occupy RAM but are never rendered. Rebalancing is too costly; they are left as dead leaves. [S4, unofficial]
*   **Boundary Artifacts:** Chunk-based methods (like H-3DGS) suffer from "chunk bleeding" or ghosting at tile boundaries. [S1, unofficial]

### Compression and Quantization (EAGLES, SOGS, LightGaussian)
These methods reduce memory footprint by quantizing attributes and pruning redundant Gaussians.

**Inputs:** Optimized 3D Gaussians $\mathcal{G}$.
**Outputs:** Compressed bitstream or quantized codebooks.

**Algorithm: Quantization and Pruning (EAGLES/SOGS)**
1.  **Importance Scoring:** Calculate an importance score for each Gaussian based on its contribution to the rendered image (e.g., opacity $\times$ projected area). [S3]
2.  **Pruning:** Remove Gaussians with the lowest importance scores (e.g., top 50% for aggressive compression). [S2, unofficial]
3.  **Quantization:**
    *   **Position/Scale:** Quantize to 16-bit half-precision or lower. [S2, unofficial]
    *   **Opacity:** Quantize to 8-bit raw logit. Note: Quantizing opacity can reduce floaters/artifacts. [S3]
    *   **Rotation:** Quantize to 8-16 bits. [S2, unofficial]
    *   **Spherical Harmonics (SH):** Factorize SH coefficients or quantize to INT6/INT8. LightGaussian uses structured pruning and SH factorization. [S1, unofficial][S3]
4.  **Grid-Based Storage (SOGS):**
    *   Arrange attributes into 2D grids.
    *   Sort grid cells and apply smoothness regularization to penalize local variance. [S3]
    *   Compress RGB grid with JPEG XL; other attributes with OpenEXR/ZIP. [S3]

**Parameters:**
*   **EAGLES:** Quantizes all attributes. Achieves 68 MB on MipNeRF360. [S3]
*   **SOGS:** Uses 2D grids. Achieves 28.80 MB on MipNeRF360. [S3]
*   **LightGaussian:** Prunes + SH factorization. Achieves >200 FPS. [S1, unofficial]

### Adaptive Filtering (Scale-Adaptive, Multi-scale)
These methods address aliasing by adjusting the Gaussian footprint during rendering or training.

**Algorithm: Scale-Adaptive Filtering (SA-GS)**
1.  **Input:** Camera parameters (resolution, distance).
2.  **Filter Calculation:** Compute a 2D low-pass filter kernel based on the current rendering resolution and camera distance. [S1, unofficial]
3.  **Rendering:** Apply the filter to the projected Gaussian footprint before rasterization. This dynamically adjusts the effective scale of the Gaussian to match the sampling frequency, preventing high-frequency artifacts. [S1, unofficial]

**Algorithm: Multi-scale Filtering**
1.  **Training:** Apply 3D smoothing regularization to constrain the frequency content of the Gaussians. [S1, unofficial]
2.  **Rendering:** Use resolution-aware selection or filtering to ensure alias-free images across sampling rates. [S1, unofficial]

### Virtual Memory and Streaming (Vmem, Budgeted GS)
These methods handle scenes larger than GPU memory.

**Algorithm: Virtual Memory (Vmem)**
1.  **Page Grouping:** Group Gaussians into fixed-size pages.
2.  **Culling:** Use a proxy mesh to determine visibility.
3.  **Streaming:** Load only visible pages into GPU buffer. [S1, unofficial]

**Algorithm: Budgeted GS (BCT)**
1.  **Training:** Train a single model with a continuous budget-quality curve.
2.  **Deployment:** Select primitives based on a target budget (primitive count) without retraining. [S1, unofficial]

## Subtopics

### [CityGaussian / VastGaussian / DoGaussian](6143-citygaussian-vastgaussian-dogaussian.md)
Block-based partitioning for city-scale scenes. Divides environments into overlapping tiles, trains independent 3DGS models per tile, and merges them at render time. Solves memory/convergence limits of monolithic training. Decouples training for distributed execution.

### [Octree-GS](7341-octree-gs.md)
Hierarchical LOD using a strict octree hierarchy for merging Gaussians. Builds upon Scaffold-GS. Sets itself apart by using octree merging rather than voxel-grid scaffolding or clustering.

### [LightGaussian](8036-lightgaussian.md)
Compression via structured pruning and Spherical Harmonic (SH) factorization. Achieves 15x size reduction and >200 FPS. Focuses on attribute-level compression (SH/position) rather than spatial clustering.

### [Scale-Adaptive Gaussian Splatting](8085-scale-adaptive-gaussian-splatting.md)
Addresses aliasing by employing an adaptive 2D low-pass filter during rendering, dynamically adjusted by resolution and camera distance. Prevents artifacts without changing the underlying Gaussian representation.

### [Multi-scale filtering](8244-multi-scale-filtering.md)
Addresses aliasing via 3D smoothing during optimization and resolution-aware filtering during rendering. Constrains frequency content to ensure alias-free images.

### [Virtual Memory for 3D Gaussian Splatting](8403-virtual-memory-for-3d-gaussian-splatting.md)
Enables rendering of scenes exceeding GPU memory by grouping Gaussians into pages, using a proxy mesh for culling, and streaming visible pages. Separates virtual and physical address spaces.

### [Gaussian clustering and merging](8436-gaussian-clustering-and-merging.md)
Reduces cost by grouping splats into clusters and replacing them with a representative Gaussian or hierarchy. Focuses on the merging primitive and culling logic.

### [Iterative Gaussian Synopsis](9289-iterative-gaussian-synopsis.md)
Top-down LOD construction. Derives coarser levels from a full-fidelity model via adaptive pruning, ensuring inter-layer consistency. Shares a foundational codebook and grids.

### [Budgeted Gaussian Splatting](9672-budgeted-gaussian-splatting.md)
Enables variable primitive counts (LOD) without retraining using a continuous budget-quality curve. Decouples LOD selection from training.

### [Hierarchical 3D Gaussian Splatting (H-3DGS)](10279-hierarchical-3d-gaussian-splatting-h-3dgs.md)
Large-scale reconstruction via chunking and merging into a global LoD hierarchy. Uses a Split Point Tree (SPT) structure. Introduces boundary artifacts like chunk bleeding.

### [Sequential Point Trees (SPTs)](10290-sequential-point-trees-spts.md)
Dynamic LoD hierarchy structure for 3DGS. Enables view-dependent LOD selection without full tree traversal. Evolves continuously during optimization.

### Hierarchical 3DGS
Cluster-based LoD method developed by Inria. Uses an octree-like tree of clusters; renders only relevant LoD. Achieves 99% reduction at far distances. Capable of interactive rendering of large scenes like Mont Saint-Michel. [S2, unofficial]

### LODGE
Level-of-detail framework for large-scale 3DGS. Dynamically selects appropriate detail levels for efficient rendering. [S1, unofficial]

### EAGLES
Quantizes all Gaussian attributes (color, position, opacity, rotation, scaling). Quantization of opacity reduces floaters/artifacts. On MipNeRF360: SSIM 0.81, PSNR 27.15, LPIPS 0.24, Size 68 MB. [S3]

### SOGS (Sparse Orthographic Grid Splatting)
Arranges attributes into 2D grids, sorts them, and applies smoothness regularization. Compresses RGB with JPEG XL, other attributes with 32-bit OpenEXR/ZIP. On MipNeRF360: SSIM 0.801, PSNR 26.98, LPIPS 0.238, Size 28.80 MB. [S3]

### 3D Gaussian Splatting Compression
Does not quantize opacity and position to avoid overlapping Gaussians. On MipNeRF360: SSIM 0.808, PSNR 27.16, LPIPS 0.228. Size unspecified. [S3]

### TRIPS
Attaches a neural feature onto Gaussians. Renders pyramid-like image feature planes according to projected Gaussian size to resolve blur issues. [S3]

### Unreachable Gaussian Handling
Strategy to tolerate cases where a child Gaussian is larger than its parent, violating the SPT cut condition. These Gaussians are left in RAM but not rendered. Rebalancing is too costly. Affects <10% of Gaussians. [S4, unofficial]

## Operating characteristics
*   **Time Complexity:** Rendering complexity is reduced from $O(N)$ to $O(\sqrt{N})$ or better for hierarchical methods, depending on the scene distribution and LOD threshold. [Parent]
*   **Memory Complexity:** Compression methods (EAGLES, SOGS) reduce storage by 8–20x. Virtual memory methods (Vmem) decouple scene size from GPU memory, allowing rendering of scenes larger than VRAM. [S1, S2, unofficial][S3]
*   **Performance:** LightGaussian achieves >200 FPS. Hierarchical 3DGS achieves 99% reduction at far distances. [S1, S2, unofficial]
*   **Strengths:** Enables real-time rendering of city-scale scenes; reduces VRAM usage; mitigates aliasing artifacts.
*   **Limitations:** Hierarchical methods may introduce boundary artifacts (chunk bleeding). Quantization may cause minor quality loss (SSIM/PSNR drop). Unreachable Gaussians waste RAM. [S1, unofficial][S3][S4, unofficial]
*   **When to Use:** Large-scale scenes, memory-constrained hardware, or when high frame rates are required.
*   **When Not to Use:** Small scenes where standard 3DGS fits in VRAM and maximum fidelity is required without compression artifacts.

## Background
3D Gaussian Splatting was introduced by Kerbl et al. (2023) as a real-time radiance field representation. The rapid adoption of 3DGS for large-scale reconstruction (e.g., cityscapes) exposed limitations in memory and training stability. Subsequent research focused on LOD and compression:
*   **Hierarchical Methods:** Inria's Hierarchical 3DGS (SIGGRAPH'24) and Octree-GS introduced tree-based LOD. [S2, unofficial]
*   **Compression:** EAGLES, SOGS, and LightGaussian (2024-2025) focused on quantization and pruning. [S3]
*   **Large-Scale Training:** CityGaussian, VastGaussian, and H-3DGS addressed training scalability via chunking. [S1, S2, unofficial]
*   **Dynamic LOD:** SPTs and Budgeted GS introduced dynamic, view-dependent LOD selection. [S4, S1, unofficial]

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S4] A LoD of Gaussians: Unified Training and Rendering for Ultra-Large Scale Reconstruction with External Memory, https://doi.org/10.48550/arXiv.2507.01110, fetched 2026-10-28 (unofficial)
