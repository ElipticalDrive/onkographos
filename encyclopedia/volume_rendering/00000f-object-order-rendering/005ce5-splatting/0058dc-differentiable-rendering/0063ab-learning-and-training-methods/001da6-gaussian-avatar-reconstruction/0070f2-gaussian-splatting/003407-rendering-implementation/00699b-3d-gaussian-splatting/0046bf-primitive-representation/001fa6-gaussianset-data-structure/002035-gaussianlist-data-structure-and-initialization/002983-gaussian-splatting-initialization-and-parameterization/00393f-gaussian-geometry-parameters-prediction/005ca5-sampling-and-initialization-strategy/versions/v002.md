# Sampling and Initialization Strategy


## Overview
Sampling and Initialization Strategy defines the methods for generating the initial set of 3D Gaussian primitives from input data (such as SfM points, NeRF models, or anchor points) before the optimization or rendering loop begins [G]. It solves the problem of establishing a valid geometric representation that covers the scene while avoiding the computational bottlenecks and failure modes of standard Structure-from-Motion (SfM) pipelines in large-scale or textureless environments [G]. It differs from sibling methods like [GSGEN](8985-gsgen.md) or [Feed-forward and Generalizable Prediction](15832-feed-forward-and-generalizable-prediction.md) by focusing on the specific geometric sampling techniques (e.g., pyramidal sampling, anchor deviation, or voxel-based decoding) rather than end-to-end generative models or feed-forward networks [G].

## Shared principles
The core mechanism involves mapping a source representation (points, voxels, or anchors) into a set of Gaussian primitives $G = \{g_i\}$, where each primitive is defined by position $\mathbf{p}_i$, opacity $\alpha_i$, scale $\mathbf{s}_i$, rotation $\mathbf{q}_i$, and color $\mathbf{c}_i$ [G].

1. **Source Discretization:** The scene is discretized into a finite set of locations [G]. Common sources include SfM point clouds, voxel grids with latent features, ray-marched points from a NeRF, and fixed anchor points [G].
2. **Attribute Prediction/Assignment:** For each location, attributes are predicted or assigned [G]. Position is often derived directly from the source location, possibly with a learned offset or deviation [G]. Scale is typically determined by the local density of points (e.g., average distance to $k$-nearest neighbors) or predicted from latent features [G]. Rotation is often aligned with surface normals if available, or initialized randomly/identity [G]. Opacity and color are predicted from latent features or assigned from source colors [G].
3. **Hierarchical or Multi-scale Construction:** Some methods construct multiple levels of Gaussians with varying densities and scales to handle multi-resolution details [S2, unofficial].

## Subtopics

### [QuickSplat](15256-quicksplat.md)
QuickSplat replaces heuristic densification with learned networks. It initializes Gaussians by discretizing the scene into sparse 3D voxels and decoding latent features via an Initializer Network. It differs from standard 3DGS by using an iterative densification-optimization loop with learned priors, handling textureless regions better than gradient-descent-only methods. Choose it for large-scale indoor scene reconstruction where SfM points are sparse or noisy.

### [Postoperative Surface Alignment](21290-postoperative-surface-alignment.md)
Postoperative Surface Alignment deforms an initial height-field Gaussian model to match postoperative MRI depth data while retaining the initial cortical texture. It differs from naive depth fitting by combining a depth loss with a transition objective to prevent surface tearing and a tracked self-supervision term to preserve appearance, using alternating optimization windows to stabilize geometry and appearance updates. Choose it when adapting a preoperative 3DGS reconstruction to postoperative anatomical changes without sparse cross-modal point correspondences.

### BrightDreamer
BrightDreamer predicts the deviation of a set of fixed anchor points to determine the centers of Gaussians [S1]. This approach avoids direct point cloud prediction from images, instead using anchor points as a structural prior [S1].

### Pyramidal Gaussian Initialization from NeRF
Pyramidal Gaussian Initialization initializes a multi-scale hierarchy of 3D Gaussians (Pyramidal Gaussians) by sampling point clouds from a trained grid-based NeRF model [S2, unofficial]. It avoids the slow and distant-failure-prone COLMAP reconstruction for large scenes [S2, unofficial].
1. Train a grid-based NeRF model $F_\Phi$ on the large-scale scene (approx. 20 minutes) to capture approximate structure [S2, unofficial].
2. Cast rays from training cameras with origin $\mathbf{o}$ and direction $\mathbf{d}$. Sample points at the estimated termination point $\hat{z}$ of each ray to generate a point cloud $P = \{(\mathbf{x}_i, \mathbf{c}_i) \mid 1 \le i \le N_P\}$, where $\mathbf{x}_i = \mathbf{o} + \hat{z}\mathbf{d}$ and $\mathbf{c}_i = F_\Phi(\mathbf{x}_i, \mathbf{d})$ [S2, unofficial].
3. Sample $L$ subsets of points from $P$ with progressively increasing frequencies. For the $l$-th subset, randomly select $N_l$ points from $P$, where $N_l = l \cdot N_1$ and $N_1$ is the number of points in the first (sparsest) subset [S2, unofficial].
4. Initialize 3D Gaussians for each subset using the standard 3DGS approach (based on average distance to nearest neighbors). This creates a hierarchy $G_{py} = \{G_l \mid 1 \le l \le L\}$ where higher levels (smaller $l$) have larger scales and lower levels have smaller scales [S2, unofficial].

### Root-Finding Initialization Strategy
Root-Finding Initialization uses a single initialization for root-finding [S3]. An alternative is to add additional initializations from the nearest 2 SMPL bones [S3]. Adding more initializations does not noticeably change metrics (Single: PSNR 31.6, SSIM 0.973, LPIPS 0.050; Two additional: PSNR 31.5, SSIM 0.972, LPIPS 0.049) but drastically increases memory and time consumption [S3]. Thus, a single initialization is preferred [S3].

### Random Initialization
Random Initialization uniformly samples triangle primitives within the bounding box of the scene [S4, unofficial]. It iteratively prunes transparent primitives during optimization [S4, unofficial]. It optimizes the random initialization to reconstruct the scene [S4, unofficial].

## Background
The standard initialization for 3D Gaussian Splatting relies on Structure-from-Motion (SfM) point clouds (e.g., from COLMAP) [G]. However, SfM fails in textureless regions and is computationally expensive for large scenes [G]. Subsequent methods introduced alternative initialization strategies: voxel-based decoding ([QuickSplat](15256-quicksplat.md)), NeRF-based sampling (PyGS) [S2, unofficial], and anchor-based deviation (BrightDreamer) [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] PyGS: Large-scale Scene Representation with Pyramidal 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2405.16829, fetched 2026-10-26 (unofficial)
[S3] ARAH: Animatable Volume Rendering of Articulated Human SDFs, https://doi.org/10.1007/978-3-031-19824-3_1, fetched 2026-10-26
[S4] DiffSoup: Direct Differentiable Rasterization of Triangle Soup for Extreme Radiance Field Simplification, https://doi.org/10.48550/arXiv.2603.27151, fetched 2026-10-30 (unofficial)
