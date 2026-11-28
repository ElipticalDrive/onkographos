# Octree-based 3D Gaussian Splatting

Also called: Cluster-based 3DGS, Hierarchical 3DGS, Octree Gaussian Splatting, Octree-GS, Octree-based 3DGS, Towards Consistent Real-time Rendering with LOD-Structured 3D Gaussians

## Overview
Octree-based 3D Gaussian Splatting (Octree-GS) is a hierarchical Level-of-Detail (LoD) method that organizes Gaussian primitives into a sparse octree structure to enable consistent real-time rendering of large-scale scenes [S7]. It addresses the memory and computational bottlenecks of standard 3DGS by dynamically selecting an appropriate subset of Gaussians based on the camera's viewing distance and scene complexity, ensuring rendering stability irrespective of trajectory changes [S7]. Unlike sibling methods such as CityGaussian or Hierarchical-GS, Octree-GS employs an explicit octree with an accumulative LoD strategy and achieves these effects in a single end-to-end training round, avoiding the post-processing overhead and stroboscopic artifacts associated with distance-interval-based selection [S7].

## How it works
### Inputs and Outputs
**Inputs:**
*   Sparse Structure-from-Motion (SfM) point cloud [S7].
*   Training images with known camera poses and intrinsics [G].
*   Hyperparameters: base voxel size $\delta$, outlier discard ratio $r_d$, visibility threshold $\tau_v$, gradient threshold $\tau_g$, growth difficulty factor $\beta$, bias increment $\epsilon$, and densification interval $T$ [S7].

**Outputs:**
*   An octree structure where each node (anchor) at level $L$ contains a set of Gaussian primitives (either explicit or neural) and a learnable LoD bias $\Delta L_j$ [S7].
*   A trained model capable of rendering novel views by selecting anchors based on distance [S7].

### Data Structures
The core data structure is a sparse octree [S7].
*   **Levels:** The octree has $K$ layers, indexed $L \in \{0, \dots, K-1\}$. LOD 0 is the coarsest level [S7].
*   **Anchors:** Each node in the octree is an "anchor." Anchors are positioned at the centers of sparse, uniform voxel grids [S7].
*   **Voxel Size:** The voxel size $V_L$ for anchors at level $L$ is defined as $V_L = \delta / 2^L$, where $\delta$ is the base voxel size for LOD 0 [S7].
*   **Attributes:** Each anchor $j$ stores:
    *   Position $\mathbf{p}_j$.
    *   Learnable LoD bias $\Delta L_j$.
    *   Associated Gaussian primitives (positions, covariances, opacities, spherical harmonics coefficients). The number of primitives per anchor is $k$ [S7].
*   **LoD Assignment:** Each anchor is assigned a discrete LoD level $L_j$. The rendering strategy is cumulative: rendering at LoD $K$ includes all primitives from levels $0$ to $K$ [S7].

### Algorithm Steps

#### 1. Anchor Initialization
1.  Calculate the distance $d_{ij}$ between each training camera center $i$ and each SfM point $j$ [S7].
2.  Determine $d_{max}$ and $d_{min}$ as the $r_d$-th largest and smallest distances, respectively, to discard outliers. Typical $r_d = 0.999$ [S7].
3.  Calculate the number of octree layers $K$:
    $$ K = \lfloor \log_2(d_{max}/d_{min}) \rceil + 1 $$
    where $\lfloor \cdot \rceil$ denotes the round operator [S7].
4.  Construct the octree-structured grids with $K$ layers [S7].
5.  Initialize anchors at the voxel centers for each layer $L$ with voxel size $V_L = \delta / 2^L$ [S7].
6.  Initialize anchor attributes (position, bias $\Delta L_j = 0$, and Gaussian primitives) [S7].

#### 2. Anchor Selection (Rendering)
For a given viewpoint $i$ and anchor $j$:
1.  Estimate the observation distance $d_{ij}$ [S7]. If camera intrinsics vary, apply a focal scale factor $s$ to adjust the distance equivalently: $d'_{ij} = d_{ij} \cdot s$ [S7].
2.  Calculate the fractional LOD level $L^*_{ij}$:
    $$ L^*_{ij} = \log_2(d_{max} / d'_{ij}) $$
3.  Apply the learnable LoD bias $\Delta L_j$ and clamp the result to $[0, K-1]$ using a clamping function $\Phi(\cdot)$:
    $$ \hat{L}_{ij} = \lfloor \Phi(L^*_{ij} + \Delta L_j) \rfloor $$
    where $\lfloor \cdot \rfloor$ is the floor function [S7].
4.  **Selection Criterion:** An anchor $j$ is selected if its assigned level $L_j \le \hat{L}_{ij}$ [S7].
5.  **Opacity Blending for Smooth Transitions:** To prevent artifacts at LoD boundaries, select "nearly satisfied" anchors where $L_j = \hat{L}_{ij} + 1$ [S7]. Scale the opacities of these nearly satisfied anchors by $(\hat{L}_{ij} - L_j)$ [S7].

#### 3. Progressive Training
Training is divided into phases to arrange Gaussians into appropriate LoD levels [S7].

**Phase 1: Initial Densification**
1.  Run for $T_{prog}$ iterations (e.g., 10k) with a progress training weight $\omega = 1.5$ [S7].
2.  Extend densification by an additional 10k iterations [S7].

**Phase 2: Adaptive Control (Every $T$ iterations, e.g., 100)**
1.  **Anchor Growing:**
    *   Calculate the average accumulated gradient $\nabla g$ of spawned Gaussian primitives [S7].
    *   If $\nabla g > \tau_g$ (default 0.0002), the primitive is significant [S7].
    *   If the primitive is in an empty voxel, convert it to a new anchor [S7].
    *   **Next-Level Growth:** To assign the LoD level $L$ of the new anchor:
        *   Calculate thresholds: $\tau_{gL} = \tau_g \cdot 2^{\beta L}$ (default $\beta = 0.2$) [S7].
        *   If $\nabla g > \tau_{g(L+1)}$, promote the anchor to level $L+1$ [S7].
        *   If $\tau_{gL} < \nabla g < \tau_{g(L+1)}$, keep the anchor at level $L$ [S7].
    *   **Bias Adjustment:** Calculate the anchor gradient $\nabla v$ (average gradient of spawned primitives for that anchor) [S7].
    *   If $\nabla v > \tau_{gL} \cdot 0.25$, increase the anchor's LoD bias: $\Delta L_j = \Delta L_j + \epsilon$ (default $\epsilon = 0.01$) [S7].

2.  **Anchor Pruning:**
    *   Compute the average opacity of Gaussians generated over the last $T$ iterations [S7].
    *   Remove anchors with low average opacity [S7].
    *   Calculate "view-frequency": the probability that an anchor is selected in training view frustums [S7].
    *   Remove anchors with view-frequency below the visibility threshold $\tau_v$ (e.g., 0.7 for small scenes, 0.01 for large scenes) [S7].

### Parameters
*   $\delta$: Base voxel size for LOD 0 [S7].
*   $r_d$: Outlier discard ratio (typical 0.999) [S7].
*   $\tau_g$: Gradient threshold for growing (default 0.0002) [S7].
*   $\beta$: Growth difficulty factor (default 0.2) [S7].
*   $\epsilon$: LoD bias increment (default 0.01) [S7].
*   $\tau_v$: Visibility threshold (0.7 small, 0.01 large, 0.2 multi-scale) [S7].
*   $T$: Densification interval (default 100) [S7].
*   $\omega$: Progress training weight (default 1.5) [S7].
*   $s$: Focal scale factor for varying intrinsics [S7].

### Edge Cases and Pitfalls
*   **Varying Intrinsics:** The observation distance must be scaled by $s$ to ensure consistent footprint sizing across different camera resolutions [S7].
*   **Stroboscopic Effects:** Unlike methods using fixed distance intervals, Octree-GS uses cumulative LoD and opacity blending to avoid stroboscopic artifacts when switching levels [S7].
*   **Floaters:** Anchors with low view-frequency (invisible in most training views) can cause floaters and must be pruned [S7].

## Operating characteristics
*   **Performance:** Octree-GS achieves up to 10x faster rendering than state-of-the-art methods in large-scale scenes [S7]. On the MatrixCity dataset, it achieves 32.0 FPS with 3.59M Gaussians, compared to 6.91 FPS with 20.8M Gaussians for Hierarchical-GS [S7].
*   **Memory Efficiency:** By dynamically selecting only relevant LoD levels, it significantly reduces the number of visible Gaussians ($N_{vis}$) and storage overhead. It achieves a 99% reduction in visible primitives at far distances [S6, unofficial].
*   **Training Time:** Due to single-stage LoD construction and fewer primitives to optimize, training time is reduced (e.g., 20-23 minutes vs 38-69 minutes on Mip-NeRF360) [S7].
*   **Quality:** Achieves higher PSNR across all scales compared to baselines, effectively handling aliasing without erosion artifacts [S7].

## Background
Octree-GS was introduced to address the limitations of existing LoD methods in 3D Gaussian Splatting, such as the need for post-processing (Hierarchical-GS), manual threshold tuning (CityGaussian), or multi-resolution point cloud inputs (LetsGo) [S7]. It builds upon the Scaffold-GS representation, extending it with an explicit octree structure and a cumulative LoD strategy [S7]. The method was developed to enable consistent real-time rendering for large-scale urban environments and complex scenes [S7].

## Subtopics

### Hierarchical 3D Gaussian Splatting (H-3DGS)
The parent method that trains chunks independently and merges them into a global LoD hierarchy. Octree-GS is a specific implementation within this family using an octree structure [10279-hierarchical-3d-gaussian-splatting-h-3dgs.md].

### CityGaussian / VastGaussian / DoGaussian
Sibling methods that divide scenes into tiles, train per tile, and merge with overlap, using a LoD pyramid for streaming. They differ from Octree-GS by requiring manual distance threshold adjustments and lacking the explicit octree accumulative strategy [6143-citygaussian-vastgaussian-dogaussian.md].

### Iterative Gaussian Synopsis
A sibling method that uses a different approach to synopsis generation, likely involving iterative refinement rather than a fixed octree hierarchy [9289-iterative-gaussian-synopsis.md].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S3] Gaussian Splatting for real-time scene reconstruction: Methods, systems, and trade-offs, https://doi.org/10.1016/j.patcog.2026.115120, fetched 2026-10-26
[S4] 3DGS.zip: A survey on 3D Gaussian Splatting Compression Methods, https://arxiv.org/abs/2407.09510v5, fetched 2026-10-26
[S5] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S6] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S7] Octree-GS: Towards Consistent Real-time Rendering with LOD-Structured 3D Gaussians, https://doi.org/10.1109/tpami.2025.3568201, fetched 2026-10-27
[S8] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
[S9] Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD, https://arxiv.org/abs/2610.03162, fetched 2026-10-29 (unofficial)
