# Pose-free and Feed-forward Training

Also called: Pose-free 3DGS, Feed-forward Gaussian Splatting, Generalizable 3DGS

## Overview
This family encompasses methods that reconstruct 3D Gaussian Splatting (3DGS) scenes directly from input images without iterative per-scene optimization or reliance on pre-computed Structure-from-Motion (SfM) pipelines [S1, unofficial]. Unlike traditional 3DGS which requires minutes to hours of per-scene optimization using known camera poses, these methods use feed-forward neural networks or geometry priors to generate primitives and camera parameters in seconds [S1, unofficial]. They are distinguished from sibling methods like [Gaussian Avatar Reconstruction](1575-gaussian-avatar-reconstruction.md) by their focus on generalizable scene reconstruction from static, unposed image collections rather than animatable human models.

## Shared principles
All methods in this family rely on differentiable rendering to compute gradients from the rendered image to the Gaussian parameters (position, covariance, opacity, spherical harmonics) and/or network weights [S1, 6136-instantsplat.md, 6464-pixelsplat.md, unofficial]. The rendering process remains differentiable with respect to the scene representation [S1, 6136-instantsplat.md, 6464-pixelsplat.md, unofficial]. Loss functions typically combine photometric reconstruction error, such as $\mathcal{L}_{L1}$ or $\mathcal{L}_{MSE}$, with perceptual or structural metrics like $\mathcal{L}_{MS-SSIM}$ or $\mathcal{L}_{LPIPS}$ [6136-instantsplat.md][6464-pixelsplat.md].

## Subtopics

### [InstantSplat](6136-instantsplat.md)
InstantSplat is a pose-free, sparse-view 3D Gaussian Splatting method that generates high-quality scene reconstructions from uncalibrated image collections in seconds by leveraging geometry priors from foundation models like MASt3R or VGGT. Unlike traditional 3D Gaussian Splatting which requires precise camera poses from Structure-from-Motion (SfM) pipelines, InstantSplat bypasses explicit pose estimation, enabling rapid training on casual or sparse captures while maintaining real-time rendering capabilities. It distinguishes itself from other pose-free feed-forward methods (e.g., [pixelSplat](6464-pixelsplat.md)) by relying on external geometric priors rather than learning pose and geometry jointly from image pairs alone, allowing it to handle sparse views more effectively.

### [pixelSplat](6464-pixelsplat.md)
pixelSplat is a feed-forward model that reconstructs a 3D radiance field parameterized by 3D Gaussian primitives from a pair of input images. It addresses the core problems of overcoming local minima inherent to sparse, locally supported representations and resolving scale ambiguity in generalizable novel view synthesis. Unlike light-field transformers which produce black-box ray fields, pixelSplat generates an explicit, editable 3D representation. It is distinguished from its siblings by its specific use of an epipolar transformer to resolve scale ambiguity and a probabilistic sampling mechanism to avoid local minima in primitive regression.

### [Joint Pose Optimization](6888-joint-pose-optimization.md)
Joint Pose Optimization (JPO) refers to methods that simultaneously optimize camera poses and the scene representation (neural or geometric) to eliminate the dependency on pre-computed, external Structure-from-Motion (SfM) pipelines. It solves the problem of novel-view synthesis and 3D reconstruction from unposed image collections where camera parameters are unknown or unreliable. Unlike sibling methods that may rely on incremental depth priors or temporal continuity, JPO typically treats pose and scene as coupled variables optimized via differentiable rendering.

### [Loss Scheduling for Candidate Head](13095-loss-scheduling-for-candidate-head.md)
A training strategy that gradually reduces the influence of an image-dependent 'candidate head' (Ĉi) during training, transitioning to a shared representation (Ĉ) to stabilize pose estimation for hard-pose images.

### AnySplat
- definition: Pose-free feed-forward network: a single forward pass over uncalibrated image collections yields 3D Gaussian primitives plus per-image intrinsics and extrinsics [S1, unofficial]. It scales to casual dense or sparse multi-view captures with no pose annotations, matching pose-aware baselines and surpassing prior pose-free methods (NoPoSplat-class) in zero-shot evaluations at real-time rendering latency [S1, unofficial].

### VS-Splat Extension for Noisy Camera Poses
- definition: An extension to handle noisy camera poses, following SHARE implementation with modified input module [S2, unofficial].
- step: Use three 1x1 2D convolutional layers to reduce channel dimension of feature maps from 768 to 128, matching SHARE input dimension. Each conv layer preceded by normalization and Swish activation [S2, unofficial].
- parameter: M (number of fine-grained primitives per voxel): 8 (due to GPU memory constraints) [S2, unofficial].
- number: On CO3D dataset, Oursext yields PSNR 21.62, SSIM 0.864, LPIPS 0.211 [S2, unofficial].

## Background
The family originates from the need to bypass the computational bottleneck of Structure-from-Motion (SfM) and per-scene optimization in standard 3D Gaussian Splatting. Early work like pixelSplat [S1, unofficial] introduced feed-forward architectures for 2-view synthesis, while later developments like InstantSplat [S1, unofficial] leveraged foundation models (MASt3R/VGGT) for pose-free sparse-view reconstruction. AnySplat [S1, unofficial] extends this to uncalibrated multi-view collections. VS-Splat [S2, unofficial] provides voxel-selective feed-forward reconstruction with extensions for noisy poses.

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] VS-Splat: Voxel-Selective feed-forward Gaussian Splatting for end-to-end 3D object reconstruction from sparse-views, https://doi.org/10.48550/arxiv.2609.12343, fetched 2026-10-26 (unofficial)
