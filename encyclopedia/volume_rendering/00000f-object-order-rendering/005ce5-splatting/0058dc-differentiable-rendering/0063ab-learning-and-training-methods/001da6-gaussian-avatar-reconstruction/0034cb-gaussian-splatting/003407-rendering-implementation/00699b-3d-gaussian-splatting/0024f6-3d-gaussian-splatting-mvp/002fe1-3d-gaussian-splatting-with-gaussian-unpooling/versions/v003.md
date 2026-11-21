# 3D Gaussian Splatting with Gaussian Unpooling

Also called: FSGS

## Overview
FSGS is the first method to explore reconstructing 3D scenes from sparse view input [S1]. It addresses insufficient initialization data by using Gaussian unpooling to expand sparse Structure-from-Motion (SfM) points into a denser set of Gaussians, enabling faithful geometry reconstruction where standard methods fail due to lack of initial coverage [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Sparse input images, an initial 3D point cloud from Structure-from-Motion (SfM), and a pre-trained 2D depth estimation network [S1].
- **Outputs:** A set of optimized 3D Gaussians for rendering input and novel views [S1].

**Data Structures:**
The method operates on the standard 3D Gaussian Splatting data structure: a list of Gaussians, each defined by a mean position $\mathbf{\mu}$, a rotation quaternion $\mathbf{q}$, a scale vector $\mathbf{s}$, an opacity $\alpha$, and spherical harmonics coefficients $\mathbf{c}$ [G]. The initial set is sparse, derived from SfM [S1].

**Steps:**
1. **Sparse Initialization:** Initialize the Gaussian set with points from the SfM reconstruction [S1].
2. **Gaussian Unpooling:** Expand the sparse set of Gaussians into a denser set by identifying and generating new Gaussians from the existing sparse ones to fill gaps in the scene representation [S1][S2, unofficial]. The exact algorithmic details of the unpooling expansion are not specified in the sources [S1].
3. **Depth Supervision:** Render depth images from the current Gaussian set [G]. Use a pre-trained 2D depth estimation network to predict depth from the input images to supervise geometry [S1].
4. **Optimization:** Optimize the Gaussian parameters (positions, scales, rotations, opacities, SH coefficients) using standard photometric rendering loss and depth supervision [S1]. Remove Gaussians with incorrect depth values during optimization [S1].
5. **Regularization:** Utilize Score Distillation Sampling (SDS) loss to encourage rendered results from novel viewpoints to be more faithful [S1].

**Formulas:**
- The exact mathematical formulations for the depth loss and SDS loss are not given in the sources [S1].

**Parameters:**
- Pre-trained 2D depth estimation network: Not specified which network [S1].
- Number of unpooling iterations or expansion factor: Not given in the sources.

**Edge Cases and Pitfalls:**
- Sparse SfM initialization may have large gaps, requiring effective unpooling to avoid holes in the reconstruction [S1].
- Incorrect depth predictions from the 2D network can mislead optimization; the method removes Gaussians with incorrect depth values to mitigate this [S1].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Enables reconstruction from sparse views, a regime where standard 3DGS fails [S1].
- **Strengths:** First method to explore sparse-view reconstruction via Gaussian unpooling [S1].
- **Limitations:** Relies on SfM for initialization, which may be unreliable in textureless or repetitive regions. Requires a pre-trained 2D depth network [S1].

## Background
FSGS was introduced as the first method to explore reconstructing 3D scenes from sparse view input using Gaussian unpooling [S1]. It builds on the 3D Gaussian Splatting framework [3D Gaussian Splatting MVP](3410-3d-gaussian-splatting-mvp.md) and addresses the limitation of requiring dense input views [G]. It is part of a family of sparse-view methods including SparseGS, CoherentGS, and DNGaussian, which also use depth inputs estimated by pre-trained 2D networks [S1].

## Subtopics

### SparseGS
Uses monocular depth and diffusion priors for ~3-view input [S2, unofficial].

### DNGaussian
Uses depth-normal regularizers [S2, unofficial].

### InstantSplat
Pose-free and sparse via MASt3R/VGGT geometry priors, trains in seconds [S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
