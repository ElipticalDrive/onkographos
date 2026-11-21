# Joint Pose Optimization

Also called: BARF, Unposed images methods

## Overview
Joint Pose Optimization (JPO) refers to methods that simultaneously optimize camera poses and the scene representation (neural fields or Gaussian splats) to eliminate the dependency on pre-computed Structure-from-Motion (SfM) pipelines [S1]. This approach relaxes the strict requirement for known camera parameters, allowing novel-view synthesis from unposed images using only photometric loss [S1]. It distinguishes itself from pose-free feed-forward methods by iteratively refining poses during training rather than predicting them in a single forward pass, though it typically requires rough initialization and is limited to small motions [S1].

## Shared principles
All methods in this family rely on differentiable rendering to compute gradients from the rendered image to the Gaussian parameters (position, covariance, opacity, spherical harmonics) and/or network weights [S1, 6136-instantsplat.md, 6464-pixelsplat.md]. The rendering process remains differentiable with respect to the scene representation [S1, 6136-instantsplat.md, 6464-pixelsplat.md]. Loss functions typically combine photometric reconstruction error, such as $\mathcal{L}_{L1}$ or $\mathcal{L}_{MSE}$, with perceptual or structural metrics like $\mathcal{L}_{MS-SSIM}$ or $\mathcal{L}_{LPIPS}$ [6136-instantsplat.md][6464-pixelsplat.md].

## Subtopics

### [1D Signal Alignment for Joint Optimization Analysis](8423-1d-signal-alignment-for-joint-optimization-analysis.md)
1D Signal Alignment is a simplified analytical model used to investigate the local convergence behavior of joint camera pose and scene reconstruction optimization. It reduces the complex 3D joint optimization problem into a 1D signal alignment task to analyze how the signal spectrum affects gradient stability. Unlike full 3D joint optimization methods which suffer from high-frequency spectral noise causing local optima, this method demonstrates that applying a Gaussian filter to the signal spectrum stabilizes gradients and ensures convergence to the global optimum within a specific initial range.

### Iterative Pose Optimization Network (IPO-Net)
IPO-Net is a generalizable 3D Gaussian Splatting network trained in a self-supervised manner without relying on pre-acquired camera poses or depth [S2]. It extracts features $F_r$ and $F_t$ from reference images $I_r$ and target image $I_t$ using a Shared Image Encoder (ResNet backbone pre-trained with DINO augmented with FPN) [S2]. It iteratively updates pose and depth for $K$ iterations using a Conv-GRU module: $T_{r \rightarrow t}^{(k)}=T_{r \rightarrow t}^{(k-1)}+\Delta T_{r \rightarrow t}$ and $D_t^{(k)}=D_t^{(k-1)}+\Delta D_t$, where $\Delta T$ and $\Delta D$ are predicted differences [S2]. It transfers pre-estimated relative poses $\mathbf {T}_{r\rightarrow t}$ between reference views and target view to relative poses $\mathbf {T}_{r\rightarrow r+1}$ between nearby reference views for Gaussians prediction [S2].

## Background
Joint pose optimization emerged to address the limitations of traditional pipelines that depend on off-the-shelf pose estimation methods, which increase inference time and fail in texture-less regions or when images lack overlap [S1]. Early works proposed joint optimization for both static [HJS∗ 23,WWX∗ 21,CL23a,LMTL21,SCW∗ 25] and dynamic settings [LGM∗ 23] [S1]. However, these methods still require rough pose initialization and are limited to small motions [S1]. Alternative solutions followed incremental approaches, leveraging mono-depth prior [BWL∗ 23] or temporal continuity [FLK∗ 24] [S1]. More recent extensions enable generalizable representations [CL23b,HJS∗ 23,XLC∗ 25], enabling joint pose prediction and representation learning from sparse image collections [S1]. Ye et al. [YLX∗ 25] demonstrated efficient pose estimation and novel-view synthesis from unposed images using only photometric loss, though limited to static scenes [S1].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S2] GGRt: Towards Pose-Free Generalizable 3D Gaussian Splatting in Real-Time, https://doi.org/10.1007/978-3-031-73209-6_19, fetched 2026-10-29
