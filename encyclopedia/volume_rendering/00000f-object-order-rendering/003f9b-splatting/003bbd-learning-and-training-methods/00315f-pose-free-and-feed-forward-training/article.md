# Pose-free and Feed-forward Training

Also called: Pose-free Gaussian Splatting, Feed-forward 3DGS

## Overview
This family encompasses methods that reconstruct 3D Gaussian primitives from images without requiring pre-computed camera poses from Structure-from-Motion (SfM) pipelines like COLMAP [S1, unofficial]. It solves the dependency on accurate initial pose estimation, enabling rapid reconstruction from casual, sparse, or uncalibrated image collections [S1, unofficial]. It is distinguished from [Joint Pose Optimization](6888-joint-pose-optimization.md) by either eliminating pose variables entirely via feed-forward networks or fixing them during primary optimization, and from [Gaussian Avatar Reconstruction](1575-gaussian-avatar-reconstruction.md) by focusing on static scene geometry rather than articulated human models.

## Shared principles
The methods in this family share the fundamental mechanism of treating volumetric data as a set of parametric primitives that are transformed and rasterized [S1, unofficial][S2, unofficial]. All variants rely on differentiable rendering to compute gradients from the rendered image to the Gaussian parameters (position, covariance, opacity, spherical harmonics) and/or network weights [S1, 6136-instantsplat.md, 6464-pixelsplat.md, unofficial]. The rendering process remains differentiable with respect to the scene representation [S1, 6136-instantsplat.md, 6464-pixelsplat.md, unofficial]. Loss functions typically combine photometric reconstruction error, such as $\mathcal{L}_{L1}$ or $\mathcal{L}_{MSE}$, with perceptual or structural metrics like $\mathcal{L}_{MS-SSIM}$ or $\mathcal{L}_{LPIPS}$ [6136-instantsplat.md][6464-pixelsplat.md].

## Subtopics

### [InstantSplat](6136-instantsplat.md)
InstantSplat is a pose-free, sparse-view 3D Gaussian Splatting method that leverages MASt3R or VGGT geometry priors to initialize and train Gaussian primitives in seconds. It solves the dependency on Structure-from-Motion (SfM) pipelines like COLMAP, enabling rapid reconstruction from casual or sparse image collections without pose annotations. Unlike feed-forward methods such as [pixelSplat](6464-pixelsplat.md) that generate Gaussians directly from images, InstantSplat uses a prior-based initialization followed by differentiable optimization, distinguishing it from joint pose optimization methods like [Joint Pose Optimization](6888-joint-pose-optimization.md) by fixing or ignoring poses during the primary training phase.

### [pixelSplat](6464-pixelsplat.md)
pixelSplat is a feed-forward model that reconstructs a 3D radiance field parameterized by 3D Gaussian primitives from a pair of input images. It solves the problem of generalizable novel view synthesis by enabling real-time, memory-efficient rendering and fast 3D reconstruction at inference time. Unlike light-field transformers which produce black-box ray fields, pixelSplat generates an explicit, editable 3D representation. It is distinguished from its siblings by its specific use of an epipolar transformer to resolve scale ambiguity and a probabilistic sampling mechanism to avoid local minima in primitive regression.

### [Joint Pose Optimization](6888-joint-pose-optimization.md)
Joint Pose Optimization refers to methods that simultaneously optimize camera poses and the neural scene representation to eliminate the dependency on pre-computed camera parameters from external Structure-from-Motion (SfM) pipelines. This approach relaxes the strict requirement for known initial poses, allowing novel-view synthesis from unposed images using only photometric loss. It distinguishes itself from sibling methods like [InstantSplat](6136-instantsplat.md) and [pixelSplat](6464-pixelsplat.md) by explicitly treating camera extrinsics as learnable variables alongside the scene geometry, rather than relying on fixed poses or feed-forward pose estimation networks.

### AnySplat
Pose-free feed-forward network: a single forward pass over uncalibrated image collections yields 3D Gaussian primitives plus per-image intrinsics and extrinsics [S1, unofficial]. It scales to casual dense or sparse multi-view captures with no pose annotations; matches pose-aware baselines and surpasses prior pose-free methods (NoPoSplat-class) in zero-shot evaluations at real-time rendering latency [S1, unofficial].

### VS-Splat Extension for Noisy Camera Poses
An extension to handle noisy camera poses, following SHARE implementation with modified input module [S2, unofficial]. It uses three 1x1 2D convolutional layers to reduce channel dimension of feature maps from 768 to 128, matching SHARE input dimension. Each conv layer is preceded by normalization and Swish activation [S2, unofficial]. Due to GPU memory constraints, the number of fine-grained primitives per voxel $M$ is set to 8 [S2, unofficial]. On the CO3D dataset, this variant ("Oursext") yields PSNR 21.62, SSIM 0.864, and LPIPS 0.211 [S2, unofficial].

## Background
The family originates from the need to bypass the computational bottleneck and failure modes of traditional SfM pipelines (e.g., COLMAP) in 3D Gaussian Splatting [S1, unofficial]. InstantSplat introduced the use of pose-free geometry priors (MASt3R/VGGT) for initialization [S1, unofficial]. pixelSplat and related feed-forward models (MVSplat, latentSplat, Splatter Image) extended this to direct 2-view to splat generation [S1, unofficial]. AnySplat further generalized this to uncalibrated multi-view collections [S1, unofficial]. The VS-Splat extension addresses robustness to noisy poses in sparse-view settings [S2, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] VS-Splat: Voxel-Selective feed-forward Gaussian Splatting for end-to-end 3D object reconstruction from sparse-views, https://doi.org/10.48550/arxiv.2609.12343, fetched 2026-10-26 (unofficial)
