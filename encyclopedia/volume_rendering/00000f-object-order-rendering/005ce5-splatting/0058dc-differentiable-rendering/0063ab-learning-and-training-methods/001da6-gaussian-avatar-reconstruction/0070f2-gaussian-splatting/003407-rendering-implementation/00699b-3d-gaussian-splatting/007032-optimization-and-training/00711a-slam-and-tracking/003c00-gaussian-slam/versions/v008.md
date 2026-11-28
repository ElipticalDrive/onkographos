# Gaussian-SLAM

Also called: Yugay et al. 2024 SLAM

## Overview
Gaussian-SLAM is a keyframe-based Simultaneous Localization and Mapping (SLAM) method that represents the scene as a collection of 3D Gaussians and employs sub-map fusion to handle large-scale environments [S2, unofficial]. It distinguishes itself from sibling methods like [SplaTAM](9193-splatam.md) and [GS-SLAM](9192-gs-slam.md) by explicitly managing disjoint sub-maps that are fused when they overlap, rather than maintaining a single monolithic global map or a fixed sliding window [S2, unofficial]. This architecture allows for scalable mapping while stabilizing localization through specific regularization techniques.

## How it works
**Inputs and Outputs:**
- **Inputs:** RGB-D image sequences (or RGB only, depending on variant configuration, though the core method is defined in the context of general SLAM inputs) and initial camera poses.
- **Outputs:** Camera trajectory, a fused 3D Gaussian map $\mathcal{G}$, and sub-map associations.

**Data Structures:**
- **Sub-map:** A subset of Gaussians $\mathcal{G}_k$ associated with a specific keyframe or local region.
- **Global Map:** A collection of sub-maps $\{\mathcal{G}_1, \dots, \mathcal{G}_M\}$ and their transformation relationships.
- **Gaussian Primitive $g_i$:** Defined by mean $\boldsymbol{\mu}_i$, covariance $\boldsymbol{\Sigma}_i$ (from rotation $R_i$ and scale $S_i$), opacity $\alpha_i$, and spherical harmonics coefficients $c_{i,lm}$ [S1].

**Algorithm Steps:**
1. **Initialization:** Initialize the first sub-map $\mathcal{G}_1$ using the first few frames. [G] Standard 3DGS initialization (e.g., from SfM points or random initialization with depth constraints) is used.
2. **Tracking:** For each new frame, optimize the camera pose $T_{CW}$ against the current active sub-map(s) to minimize reprojection error. [S1]
3. **Mapping (Local Optimization):**
   - Insert new Gaussians in regions with poor coverage in the current sub-map.
   - Refine existing Gaussians in the sub-map using the new frame's data.
   - Apply scale regularization to encourage isotropic Gaussians, stabilizing the optimization [S1].
4. **Sub-map Management:**
   - If the current sub-map reaches a size limit or coverage threshold, finalize it and start a new sub-map $\mathcal{G}_{k+1}$.
   - Maintain a list of active sub-maps for tracking.
5. **Sub-map Fusion:**
   - Detect overlap between existing sub-maps (e.g., via feature matching or pose graph optimization).
   - When overlap is detected, fuse the overlapping sub-maps into a single coherent map by aligning their coordinate frames and merging Gaussians (handling duplicates via opacity/size pruning). [S2, unofficial]
6. **Loop Closure:** If a loop is detected (revisiting a previously mapped area), correct drift by adjusting the poses and Gaussians in the affected sub-maps and re-fusing. [S2, unofficial]

**Loss Function:**
The optimization minimizes a combined loss:
$$ L = L_{rgb} + \lambda_{depth} L_{depth} + \lambda_{scale} L_{scale} + L_{reg} $$
- $L_{rgb}$: Photometric loss (e.g., $L_1$ or SSIM) between rendered and observed RGB.
- $L_{depth}$: Depth loss if depth is available.
- $L_{scale}$: Scale regularization loss. This term penalizes anisotropic scales to encourage isotropic Gaussians, which stabilizes localization and mapping [S1].
- $L_{reg}$: Other regularization terms (e.g., opacity regularization).

**Parameters:**
- $\lambda_{scale}$: Weight for scale regularization. Not given in the sources.
- Keyframe interval: Not given in the sources.
- Sub-map size limit: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** Sub-map fusion requires accurate pose alignment; errors in initial pose estimation can lead to misaligned Gaussians.
- **Boundaries:** Handling the boundary between sub-maps during fusion is critical to avoid artifacts. [G] Duplicate Gaussians with high overlap should be merged or pruned based on opacity.
- **Degenerate Input:** Rapid motion or textureless regions can cause tracking failure. [G] Robust tracking requires fallback mechanisms or keyframe selection strategies.

## Subtopics

### MonoGS
MonoGS is a monocular RGB-only SLAM method that performs online optimization by mapping the scene as 3D Gaussian Splatting (3DGS) primitives [S2, unofficial]. It restricts the joint estimation of camera poses and Gaussian map parameters to a selected set of keyframes, enabling real-time tracking and mapping from a single camera [S1]. Unlike RGB-D SLAM methods that rely on depth sensors, MonoGS infers geometry and pose solely from photometric consistency across keyframes [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time complexity depends on the number of Gaussians $N$ and the number of keyframes. Rendering is $O(N)$ per pixel, but optimization is iterative.
- Memory complexity is $O(N)$ for storing Gaussian parameters. Sub-map fusion adds overhead for managing multiple maps.

**Measured Performance:**
- On the Replica dataset, Gaussian-SLAM achieves PSNR 37.50, SSIM 0.96, and LPIPS 0.07 [S1].
- Note: The table in [S1] lists "Gaussian-SLAM [183]" with these metrics, but also lists "GaussianSplattingSLAM [182]" with PSNR 38.90. The specific performance of Gaussian-SLAM is cited as PSNR 37.50 [S1].

**Scaling:**
- Sub-map fusion allows scaling to larger environments by limiting the size of individual optimization problems.

**Strengths and Limitations:**
- **Strengths:** Scalable to large scenes via sub-map fusion; stable localization due to scale regularization [S1].
- **Limitations:** Requires sub-map fusion logic, which adds complexity; performance may degrade if sub-map alignment is inaccurate.

**When to use it and when not:**
- Use when mapping large-scale environments where a single global map is too large or unstable.
- Do not use if real-time performance on edge hardware is critical and sub-map fusion overhead is prohibitive.

## Background
Gaussian-SLAM was introduced by Yugay et al. in 2024 [S2, unofficial]. It is part of the wave of Gaussian Splatting SLAM methods that emerged in 2024, leveraging the efficiency and quality of 3D Gaussian Splatting for simultaneous localization and mapping [S4, unofficial]. The method addresses the limitations of previous NeRF-based SLAM systems by providing a more explicit and editable geometric representation [S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S4] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S5] Gaussian Splatting for real-time scene reconstruction: Methods, systems, and trade-offs, https://doi.org/10.1016/j.patcog.2026.115120, fetched 2026-10-26
