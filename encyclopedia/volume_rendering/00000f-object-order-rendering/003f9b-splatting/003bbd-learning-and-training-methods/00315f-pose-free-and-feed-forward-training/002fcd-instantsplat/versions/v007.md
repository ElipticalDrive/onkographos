# InstantSplat

Also called: MASt3R/VGGT-guided 3DGS, Pose-free 3D Gaussian Splatting

## Overview
InstantSplat is a pose-free, sparse-view 3D Gaussian Splatting method that trains in seconds by leveraging MASt3R or VGGT geometry priors [S1, unofficial]. Unlike standard 3D Gaussian Splatting which requires Structure-from-Motion (SfM) for camera poses, or feed-forward methods like [pixelSplat](6464-pixelsplat.md) that generate splats directly from images without explicit 3D geometry alignment, InstantSplat uses a two-stage process: first estimating sparse, accurate 3D points from uncalibrated images using a foundation model, and then optimizing Gaussian primitives against these points and the input images [S1, unofficial]. It is designed for casual, sparse, or uncalibrated captures where traditional SfM pipelines fail or are too slow.

## How it works
**Inputs:** A set of $N$ uncalibrated images $\{I_i\}_{i=1}^N$ (sparse view or casual capture). No camera intrinsics or extrinsics are required as input.
**Outputs:** A set of 3D Gaussian primitives $\mathcal{G} = \{G_j\}_{j=1}^M$, where each $G_j$ is defined by position $\mathbf{p}_j$, covariance matrix $\mathbf{\Sigma}_j$ (or rotation $\mathbf{R}_j$ and scale $\mathbf{s}_j$), opacity $\alpha_j$, and spherical harmonics coefficients $\mathbf{c}_j$.

**Data Structures:**
1.  **Point Cloud $\mathcal{P}$:** A set of 3D points $\{\mathbf{x}_k\}$ estimated from the geometry prior.
2.  **Gaussian Primitives:** Standard 3DGS representation.

**Steps:**
1.  **Geometry Prior Estimation:** Use a foundation model (MASt3R or VGGT) to estimate sparse 3D points and relative camera poses from the input images $\{I_i\}$. These models provide metric or semi-metric depth and correspondences without iterative SfM [S1, unofficial].
2.  **Initialization:** Initialize the Gaussian primitives $\mathcal{G}$ using the estimated points $\mathcal{P}$. The number of Gaussians $M$ is typically determined by the density of $\mathcal{P}$ or a fixed initialization strategy.
3.  **Differentiable Rendering:** Render the image $\hat{I}_i$ for each input view $I_i$ using the current Gaussian set $\mathcal{G}$ and the estimated camera poses from step 1. The rendering equation is the standard 3DGS alpha-blending:
    $$ \hat{C}_i = \sum_{j \in \mathcal{O}_i} \alpha_j \prod_{k=1}^{j-1} (1 - \alpha_k) \mathbf{c}_j $$
    where $\mathcal{O}_i$ is the set of Gaussians sorted by depth in view $i$.
4.  **Optimization:** Optimize the Gaussian parameters $\mathcal{G}$ and potentially refine the camera poses using a loss function combining photometric and geometric terms:
    $$ \mathcal{L} = \lambda_{img} \mathcal{L}_{L1}(\hat{I}_i, I_i) + \lambda_{ssim} \mathcal{L}_{MS-SSIM}(\hat{I}_i, I_i) + \lambda_{geo} \mathcal{L}_{geo}(\mathcal{G}, \mathcal{P}) $$
    where $\mathcal{L}_{geo}$ encourages Gaussians to align with the prior points $\mathcal{P}$ [S1, unofficial].
5.  **Training:** Train for a short duration (seconds) to converge, leveraging the high-quality initialization from the geometry prior.

**Parameters:**
*   Training time: Seconds [S1, unofficial].
*   Geometry Prior: MASt3R or VGGT [S1, unofficial].
*   Loss Weights: Standard 3DGS weights ($\lambda_{L1}, \lambda_{SSIM}$) plus a geometric regularization weight $\lambda_{geo}$ [S1, unofficial].

**Edge Cases:**
*   **Degenerate Input:** If the geometry prior fails to provide sufficient coverage (e.g., textureless regions), the geometric loss may not constrain the Gaussians adequately, leading to artifacts.
*   **Scale Ambiguity:** If the prior provides only relative depth (e.g., from monocular models), the global scale may be ambiguous. InstantSplat typically relies on the prior's metric capability or resolves scale during optimization if metric depth is available [S1, unofficial].

## Subtopics

### MoSca
A prior-aided monocular dynamic Gaussian Splatting method using CoTracker, VGG-T, DUSt3R, or metric-depth priors [S1, unofficial]. It extends the pose-free concept to dynamic scenes by incorporating temporal tracking and depth priors for monocular input.

### Shape-of-Motion
A prior-aided monocular dynamic Gaussian Splatting method using CoTracker, VGG-T, DUSt3R, or metric-depth priors [S1, unofficial]. Similar to MoSca, it focuses on reconstructing dynamic scenes from monocular video using geometric and motion priors.

### Feature-3DGS
A method storing per-Gaussian language features (CLIP, LSeg) for open-vocabulary 3D segmentation [S1, unofficial]. It integrates semantic features into the Gaussian representation, allowing for language-based queries and segmentation.

### Holistic Gaussian Initialization
A strategy to generate dense and accurate initialized 3D Gaussians within seconds for endoscopic scenes, replacing the time-consuming and sparse Structure-from-Motion (SfM) pipeline (e.g., COLMAP) used in standard 3DGS. It works for both binocular and monocular input sequences [S2, unofficial].
*   **Binocular Input:** Uses a stereo depth estimation model to predict metric depth maps $\{D_i\}$ for left views. Re-projects pixels of each left image $I_i$ into world coordinates to obtain a partial point cloud $P_i$:
    $$ \mathbf{P}_i = \mathbf{K}^{-1}\mathbf{T}_i\mathbf{D}_i(\mathbf{I}_i \odot \mathbf{M}_i) $$
    where $\odot$ is element-wise product, $\mathbf{M}_i$ is a binary mask filtering out surgical tools, and $\mathbf{K}, \mathbf{T}_i$ are camera intrinsic and extrinsic parameters [S2, unofficial].
*   **Monocular Input:** Uses a monocular depth estimation model to predict relative depth maps $\{D_i\}$. Applies the same re-projection and combination as binocular input. The scale loss in relative depth is compensated by explicit geometric constraints from real poses and rendered images during Gaussian optimization [S2, unofficial].

### Fuse-and-Refine Module
A module that enhances existing feed-forward models by merging and refining Gaussian primitives in a canonical 3D space using a hybrid Splat-Voxel representation. It reduces redundancy and handles dynamic scenes via temporal fusion [S3].
*   **Steps:**
    1.  Input: Initial set of Gaussian primitives from existing pixel-aligned feed-forward models (from current frame or warped from previous frames) [S3].
    2.  Splat-to-Voxel Transfer: Aggregate local features from Gaussian primitives into a voxel representation [S3].
    3.  Construct a coarse-to-fine voxel hierarchy by adaptively sparsifying the high-resolution voxel grid [S3].
    4.  Apply a sparse voxel transformer to features at the coarse resolution to capture global context [S3].
    5.  Generate refined Gaussian primitives at the high-resolution level from the processed voxel features [S3].
*   **Parameters:** Processing time: 15 ms to merge 200K Gaussian primitives [S3].
*   **Performance:** PSNR improvement of around 2 dB compared to pixel-aligned reconstruction approaches [S3].

## Operating characteristics
*   **Time Complexity:** Training time is in seconds, significantly faster than standard 3DGS which takes minutes to hours [S1, unofficial]. The Fuse-and-Refine module processes 200K primitives in 15 ms [S3].
*   **Memory Complexity:** Depends on the number of Gaussians initialized from the prior. Generally comparable to standard 3DGS but may require more initial points if the prior is dense.
*   **Strengths:** No need for SfM; works with sparse, casual, or uncalibrated images; very fast training; robust initialization from foundation models [S1, unofficial].
*   **Limitations:** Relies on the quality of the geometry prior (MASt3R/VGGT); may struggle with highly dynamic scenes without additional temporal priors (unless using variants like MoSca) [S1, unofficial].
*   **When to Use:** Sparse-view reconstruction, casual capture, endoscopic scenes (with Holistic Initialization), or when SfM fails.
*   **When Not to Use:** When high-precision metric poses are required and foundation models are insufficient, or for dense, multi-view captures where standard 3DGS is more efficient.

## Background
InstantSplat was introduced as a pose-free alternative to standard 3D Gaussian Splatting, leveraging recent advances in foundation models for geometry estimation (MASt3R, VGGT) [S1, unofficial]. It addresses the bottleneck of SfM in 3DGS, particularly for sparse or uncalibrated inputs. The method builds on the differentiable rendering framework of 3DGS [S1, unofficial]. Variants like Holistic Gaussian Initialization target specific domains like endoscopy [S2, unofficial], while Fuse-and-Refine extends feed-forward methods to dynamic scenes [S3].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] EndoGaussian: Real-time Gaussian Splatting for Dynamic Endoscopic Scene Reconstruction, https://huggingface.co/papers/2401.12561, fetched 2026-10-30 (unofficial)
[S3] Learning Efficient Fuse-and-Refine for Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2503.14698v2, fetched 2026-10-27
