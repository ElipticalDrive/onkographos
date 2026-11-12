# InstantSplat

Also called: MASt3R/VGGT-guided 3DGS, Pose-free 3D Gaussian Splatting

## Overview
InstantSplat is a pose-free, sparse-view 3D Gaussian Splatting method that reconstructs scenes from uncalibrated image collections without requiring camera pose annotations [S1, unofficial]. It differs from standard 3DGS by eliminating the Structure-from-Motion (SfM) preprocessing step, instead relying on geometry priors from foundation models like MASt3R or VGGT to initialize Gaussians, enabling training in seconds [S1, unofficial]. Unlike feed-forward methods such as [pixelSplat](6464-pixelsplat.md) that generate Gaussians directly from images, InstantSplat uses a differentiable optimization loop guided by these priors, bridging the gap between sparse-view optimization and pose-free efficiency.

## How it works
**Inputs:** A set of uncalibrated images (sparse or casual capture) with no known intrinsics or extrinsics.
**Outputs:** A set of 3D Gaussian primitives (position $\mu$, covariance $\Sigma$, opacity $\alpha$, spherical harmonics coefficients $c$) and optionally per-image intrinsics/extrinsics if the variant supports it.

**Data Structures:**
The scene is represented as a list of $N$ Gaussian primitives. Each primitive $i$ stores:
- $\mu_i \in \mathbb{R}^3$: Center position.
- $\Sigma_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix, typically parameterized by a rotation matrix $R_i \in SO(3)$ and a scaling vector $s_i \in \mathbb{R}^3$ such that $\Sigma_i = R_i S_i S_i^T R_i^T$.
- $\alpha_i \in [0, 1]$: Opacity.
- $c_{i, lm}$: Spherical harmonics coefficients for view-dependent color.

**Steps:**
1. **Geometry Prior Extraction:** Use a foundation model (MASt3R or VGGT) to extract dense point clouds or depth maps from the input images. These models provide metric or relative depth and feature matches without explicit pose estimation [S1, unofficial].
2. **Gaussian Initialization:** Initialize the Gaussian positions $\mu_i$ using the points from the geometry prior. If metric depth is available, re-project pixels to world coordinates. If relative depth is used, initialize positions and rely on optimization to resolve scale [S2, S1, unofficial].
3. **Differentiable Rendering:** Render the current Gaussian set to images using the standard 3DGS differentiable rasterizer. This requires estimating camera parameters (intrinsics $K$, extrinsics $T$) for each input view, either as learnable parameters or predicted by the network [S1, unofficial].
4. **Loss Computation:** Compute the loss between rendered images and input images. The loss typically combines photometric error ($\mathcal{L}_{L1}$ or $\mathcal{L}_{MSE}$) and perceptual metrics ($\mathcal{L}_{MS-SSIM}$) [S1, 6136-instantsplat.md, unofficial].
5. **Optimization:** Update Gaussian parameters (position, covariance, opacity, SH) and camera parameters via gradient descent. The geometry priors act as a regularizer or initialization constraint, preventing the optimization from diverging in the absence of known poses [S1, unofficial].
6. **Convergence:** Iterate until the loss converges or a fixed number of steps is reached. The method is designed to converge in seconds due to the high-quality initialization [S1, unofficial].

**Parameters:**
- Learning rate: Standard 3DGS values (e.g., $1.6 \times 10^{-4}$ for positions, $1.0 \times 10^{-5}$ for covariances) [G].
- Number of optimization steps: Typically very few (seconds of training) compared to standard 3DGS which may take minutes [S1, unofficial].
- Geometry Prior Model: MASt3R or VGGT [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Scale Ambiguity:** When using relative depth priors (e.g., from monocular models), the global scale is ambiguous. The optimization must resolve this using geometric constraints from rendered images and pose consistency [S2, unofficial].
- **Sparse Views:** With very few views, the geometry prior may be noisy. The method relies on the robustness of MASt3R/VGGT to provide sufficient coverage [S1, unofficial].
- **Precision:** Standard float32 precision is sufficient for the optimization loop [G].

## Subtopics

### MoSca
A prior-aided monocular dynamic Gaussian Splatting method using CoTracker, VGG-T, DUSt3R, or metric-depth priors [S1, unofficial]. It extends InstantSplat's pose-free philosophy to dynamic scenes by leveraging tracking and depth priors for temporal consistency.

### Shape-of-Motion
A prior-aided monocular dynamic Gaussian Splatting method using CoTracker, VGG-T, DUSt3R, or metric-depth priors [S1, unofficial]. Similar to MoSca, it focuses on reconstructing dynamic scenes from monocular input without explicit pose estimation, using foundation model priors for geometry and motion.

### Feature-3DGS
A method storing per-Gaussian language features (CLIP, LSeg) for open-vocabulary 3D segmentation [S1, unofficial]. It integrates semantic information into the Gaussian representation, allowing for semantic queries on the reconstructed scene.

### Holistic Gaussian Initialization
A strategy to generate dense and accurate initialized 3D Gaussians within seconds for endoscopic scenes, replacing the time-consuming SfM pipeline (e.g., COLMAP) [S2, unofficial]. It works for both binocular and monocular input sequences.
- **Binocular Input:** Uses a stereo depth estimation model to predict metric depth maps $D_i$ for left views. Re-projects pixels of each left image $I_i$ into world coordinates to obtain partial point cloud $P_i$ using:
  $$ P_i = K^{-1} T_i D_i(I_i \odot M_i) $$
  where $\odot$ is element-wise product, $M_i$ is a binary mask filtering out surgical tools, and $K, T_i$ are known camera intrinsic and extrinsic parameters [S2, unofficial].
- **Monocular Input:** Uses a monocular depth estimation model to predict relative depth maps $D_i$. Applies the same re-projection and combination as binocular input. The scale loss in relative depth is compensated by explicit geometric constraints from real poses and rendered images during Gaussian optimization [S2, unofficial].

### Fuse-and-Refine Module
A module that enhances existing feed-forward models by merging and refining Gaussian primitives in a canonical 3D space using a hybrid Splat-Voxel representation [S3]. It reduces redundancy and handles dynamic scenes via temporal fusion.
- **Input:** Initial set of Gaussian primitives from existing pixel-aligned feed-forward models (from current frame or warped from previous frames) [S3].
- **Splat-to-Voxel Transfer:** Aggregates local features from Gaussian primitives into a voxel representation [S3].
- **Coarse-to-Fine Hierarchy:** Constructs a coarse-to-fine voxel hierarchy by adaptively sparsifying the high-resolution voxel grid [S3].
- **Sparse Voxel Transformer:** Applies a sparse voxel transformer to features at the coarse resolution to capture global context [S3].
- **Refinement:** Generates refined Gaussian primitives at the high-resolution level from the processed voxel features [S3].
- **Parameters:** Processing time is 15 ms for merging 200K Gaussian primitives [S3].
- **Performance:** Delivers a PSNR improvement of around 2 dB compared to pixel-aligned reconstruction approaches [S3].

## Operating characteristics
**Time Complexity:** Training time is significantly reduced compared to standard 3DGS, often completing in seconds due to the high-quality initialization from geometry priors [S1, unofficial]. The Fuse-and-Refine module processes 200K primitives in 15 ms [S3].
**Memory Complexity:** Memory usage is comparable to standard 3DGS, proportional to the number of Gaussian primitives. The hybrid Splat-Voxel representation in Fuse-and-Refine adds overhead for voxel storage but reduces redundancy [S3].
**Performance:** InstantSplat matches pose-aware baselines and surpasses prior pose-free methods in zero-shot evaluations at real-time rendering latency [S1, unofficial]. The Fuse-and-Refine module achieves interactive rates (15fps with 350ms delay) on a single H100 GPU [S3].
**Strengths:** No need for SfM preprocessing; works with sparse or casual captures; fast training; handles dynamic scenes with extensions.
**Limitations:** Relies on the quality of foundation model priors (MASt3R/VGGT); may struggle with highly ambiguous scenes where priors are insufficient [S1, unofficial].

## Background
InstantSplat was introduced as a pose-free alternative to standard 3D Gaussian Splatting, addressing the bottleneck of Structure-from-Motion (SfM) preprocessing [S1, unofficial]. It leverages recent advances in foundation models like MASt3R and VGGT to provide geometry priors, enabling rapid reconstruction from uncalibrated images [S1, unofficial]. The method is part of the broader trend towards pose-free and feed-forward training methods in 3D Gaussian Splatting [S1, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] EndoGaussian: Real-time Gaussian Splatting for Dynamic Endoscopic Scene Reconstruction, https://huggingface.co/papers/2401.12561, fetched 2026-10-30 (unofficial)
[S3] Learning Efficient Fuse-and-Refine for Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2503.14698v2, fetched 2026-10-27
