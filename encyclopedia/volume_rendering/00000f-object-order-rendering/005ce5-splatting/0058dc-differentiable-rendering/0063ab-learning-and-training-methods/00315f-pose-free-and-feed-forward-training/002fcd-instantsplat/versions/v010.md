# InstantSplat

Also called: MASt3R/VGGT-guided 3DGS, Pose-free 3D Gaussian Splatting

## Overview
InstantSplat is a pose-free, sparse-view 3D Gaussian Splatting method that eliminates the need for explicit camera pose estimation by leveraging geometry priors from foundation models like MASt3R or VGGT [S1, unofficial]. It enables training in seconds by directly initializing Gaussians from these priors, distinguishing itself from standard 3DGS which requires time-consuming Structure-from-Motion (SfM) pipelines, and from feed-forward methods like [pixelSplat](6464-pixelsplat.md) which generate Gaussians directly from images without iterative optimization [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A sparse set of uncalibrated images (no camera poses required).
- **Output:** A set of 3D Gaussian primitives (position, covariance, opacity, spherical harmonics) optimized for the input views.

**Data Structures:**
The method utilizes the standard 3D Gaussian representation:
- Position: $\mathbf{\mu} \in \mathbb{R}^3$
- Covariance: $\mathbf{\Sigma} \in \mathbb{R}^{3 \times 3}$ (often represented as rotation $\mathbf{R}$ and scale $\mathbf{s}$)
- Opacity: $\alpha \in [0, 1]$
- Color: Spherical Harmonics coefficients $\mathbf{c}_{lm}$

**Steps:**
1. **Geometry Prior Extraction:** Use a foundation model (MASt3R or VGGT) to extract dense depth maps and relative camera poses from the input image set [S1, unofficial]. These models provide metric-scale depth and consistent relative geometry without explicit SfM optimization.
2. **Gaussian Initialization:** Back-project pixels from the input images into 3D space using the estimated depth and relative poses to create a dense point cloud [S1, unofficial]. This point cloud serves as the initial positions for the 3D Gaussians.
3. **Optimization:** Perform differentiable rendering optimization to refine the Gaussian parameters (position, covariance, opacity, SH coefficients) to minimize photometric and perceptual losses [S1, 6136-instantsplat.md, unofficial]. The rendering process remains differentiable with respect to the scene representation [S1, 6136-instantsplat.md, 6464-pixelsplat.md, unofficial].

**Loss Functions:**
The loss typically combines photometric reconstruction error and perceptual metrics [6136-instantsplat.md][6464-pixelsplat.md]:
$$ \mathcal{L} = \mathcal{L}_{L1} + \lambda \mathcal{L}_{MS-SSIM} $$
where $\mathcal{L}_{L1}$ is the L1 loss between rendered and ground truth images, and $\mathcal{L}_{MS-SSIM}$ is the multi-scale structural similarity loss [6136-instantsplat.md][6464-pixelsplat.md].

**Parameters:**
- Training time: Seconds [S1, unofficial].
- View count: Sparse views (typically few-shot) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Depth Accuracy:** The quality of initialization depends heavily on the accuracy of the foundation model's depth estimation. Poor depth estimates can lead to incorrect Gaussian positions, requiring more optimization steps or resulting in artifacts [S1, unofficial].
- **Relative Pose Consistency:** While absolute poses are not required, the relative poses provided by the prior must be consistent across views to avoid geometric distortions [S1, unofficial].

## Subtopics

### [Holistic Gaussian Initialization](9776-holistic-gaussian-initialization.md)
Holistic Gaussian Initialization (HGI) is a strategy to generate dense and accurate initialized 3D Gaussians within seconds for endoscopic scenes, replacing the time-consuming and sparse Structure-from-Motion (SfM) pipeline (e.g., COLMAP) used in standard 3DGS. It works for both binocular and monocular input sequences by leveraging depth estimation models to back-project pixels into world coordinates, creating a dense point cloud that serves as the initial state for Gaussian optimization. It sets itself apart from standard 3DGS by eliminating the need for sparse SfM points, thereby accelerating initialization and improving density. See [Holistic Gaussian Initialization](9776-holistic-gaussian-initialization.md).

### [ReSplat](11511-resplat.md)
ReSplat is a feed-forward recurrent Gaussian splatting model that iteratively refines 3D Gaussian primitives without explicitly computing gradients, using rendering error as a feedback signal to guide updates. It operates in a compact subsampled 3D space to reduce the Gaussian count while improving performance and generalization. Unlike single-step feed-forward models, ReSplat leverages a weight-sharing recurrent network for multi-step refinement, enabling robust generalization across diverse datasets, view counts, and resolutions. See [ReSplat](11511-resplat.md).

### MoSca
MoSca is a prior-aided monocular dynamic Gaussian Splatting method using CoTracker, VGG-T, DUSt3R, or metric-depth priors [S1, unofficial].

### Shape-of-Motion
Shape-of-Motion is a prior-aided monocular dynamic Gaussian Splatting method using CoTracker, VGG-T, DUSt3R, or metric-depth priors [S1, unofficial].

### Feature-3DGS
Feature-3DGS is a method storing per-Gaussian language features (CLIP, LSeg) for open-vocabulary 3D segmentation [S1, unofficial].

### Fuse-and-Refine Module
The Fuse-and-Refine Module is a module that enhances existing feed-forward models by merging and refining Gaussian primitives in a canonical 3D space using a hybrid Splat-Voxel representation. It reduces redundancy and handles dynamic scenes via temporal fusion [S2].
- **Step:** Input: Initial set of Gaussian primitives from existing pixel-aligned feed-forward models (from current frame or warped from previous frames) [S2].
- **Step:** Perform Splat-to-Voxel Transfer: Aggregate local features from Gaussian primitives into a voxel representation [S2].
- **Step:** Construct a coarse-to-fine voxel hierarchy by adaptively sparsifying the high-resolution voxel grid [S2].
- **Step:** Apply a sparse voxel transformer to features at the coarse resolution to capture global context [S2].
- **Step:** Generate refined Gaussian primitives at the high-resolution level from the processed voxel features [S2].
- **Parameter:** Processing time: The module merges 200K Gaussian primitives in 15 ms [S2].
- **Number:** Quality improvement: PSNR improvement of around 2 dB compared to pixel-aligned reconstruction approaches [S2].

## Operating characteristics
- **Time Complexity:** Training time is significantly reduced to seconds compared to standard 3DGS which requires minutes to hours [S1, unofficial].
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** InstantSplat matches pose-aware baselines and surpasses prior pose-free methods in zero-shot evaluations at real-time rendering latency [S1, unofficial].
- **Scaling:** Scales to casual dense or sparse multi-view captures with no pose annotations [S1, unofficial].
- **Strengths:** Eliminates the need for explicit camera pose estimation; fast training; works with sparse views.
- **Limitations:** Relies on the accuracy of foundation model priors; may struggle with highly dynamic scenes without additional temporal priors.
- **When to use:** When camera poses are unavailable or unreliable, and fast training is required.
- **When not to use:** When high-precision absolute poses are required, or when the scene is highly dynamic and foundation model priors are insufficient.

## Background
InstantSplat was introduced as a pose-free, sparse-view 3D Gaussian Splatting method that leverages geometry priors from foundation models like MASt3R or VGGT [S1, unofficial]. It addresses the limitation of standard 3DGS which requires time-consuming Structure-from-Motion (SfM) pipelines for camera pose estimation [S1, unofficial]. The method enables training in seconds by directly initializing Gaussians from these priors, making it suitable for applications where rapid reconstruction from sparse, uncalibrated images is needed [S1, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] Learning Efficient Fuse-and-Refine for Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2503.14698v2, fetched 2026-10-27
