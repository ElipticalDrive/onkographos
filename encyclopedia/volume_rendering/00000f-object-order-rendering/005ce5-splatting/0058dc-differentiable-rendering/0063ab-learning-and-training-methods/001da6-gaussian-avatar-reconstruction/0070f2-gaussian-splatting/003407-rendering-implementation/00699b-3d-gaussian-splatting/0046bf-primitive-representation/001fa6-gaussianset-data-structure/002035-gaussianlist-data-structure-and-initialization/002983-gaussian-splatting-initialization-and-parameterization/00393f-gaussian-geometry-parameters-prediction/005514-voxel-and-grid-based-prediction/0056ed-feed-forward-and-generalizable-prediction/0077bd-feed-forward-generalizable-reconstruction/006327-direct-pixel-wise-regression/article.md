# Direct Pixel-Wise Regression


## Overview
Direct Pixel-Wise Regression is a feed-forward method that predicts a set of 3D Gaussian primitives directly from input image patches, establishing a one-to-one correspondence between 2D pixels and 3D Gaussians [S1]. It differs from sibling methods like [Triplane-LRM](16746-feed-forward-generalizable-reconstruction.md) by avoiding fixed-resolution latent representations (e.g., triplanes), thereby enabling pixel-aligned prediction of per-Gaussian colors and scaling Gaussian count with input resolution to capture high-frequency details [S1].

## Shared principles
See [Feed-forward Generalizable Reconstruction](16746-feed-forward-generalizable-reconstruction.md) for shared principles.

## Subtopics

### [FineSplat](15257-finesplat.md)
[FineSplat](15257-finesplat.md) is a feed-forward 3D Gaussian Splatting framework that predicts a set of 3D Gaussians from $N$ context views using a two-stage dual-grouping strategy to balance local detail preservation with global multi-view reasoning. It distinguishes itself from sibling methods like [YoNoSplat](16425-yonosplat.md) and [ZipSplat](16425-zipsplat.md) by employing high-resolution 4x4 patch tokens instead of coarser 14x14 patches, combined with serialized attention for 3D spatial grouping to avoid the prohibitive cost of global multi-view attention.

### [SparSplat](15813-sparsplat.md)
[SparSplat](15813-sparsplat.md) is a generalizable, feed-forward model that maps a sparse set of input images to 2D Gaussian Splatting (2DGS) parameters in a single forward pass to perform both 3D shape reconstruction and novel view synthesis. It differs from its sibling [MVSGaussian](16571-mvsgaussian.md) by using 2D planar primitives instead of 3D volumetric primitives, which ensures multi-view depth consistency required for coherent TSDF mesh fusion. Unlike prior generalizable methods designed primarily for novel view synthesis, SparSplat is explicitly optimized for feed-forward surface reconstruction, offering inference speeds nearly two orders of magnitude faster than implicit volume rendering methods.

### [Generalizable Pixel-Wise 3D Gaussian Splatting](17059-generalizable-pixel-wise-3d-gaussian-splatting.md)
[Generalizable Pixel-Wise 3D Gaussian Splatting](17059-generalizable-pixel-wise-3d-gaussian-splatting.md) (GPS-Gaussian) is a feed-forward method for real-time novel view synthesis of humans from sparse camera views. It solves the problem of per-subject optimization latency in 3D Gaussian Splatting by directly regressing Gaussian parameters from source images without fine-tuning. Unlike sibling methods that recompute view-correlated modules for every target viewpoint, GPS-Gaussian performs source-view computation once and renders novel views instantly, allowing concurrent rendering of multiple views.

### GS-LRM / Pixel-Aligned Gaussian Prediction
GS-LRM predicts 3D Gaussians directly from image patches, establishing a one-to-one correspondence between 2D pixels and 3D Gaussians [S1]. The Gaussian center is calculated as $xyz = rayo + t \cdot rayd$, where $t$ is the ray distance, $rayo$ is the ray origin, and $rayd$ is the ray direction [S1]. Each Gaussian is parameterized by $q=12$ values: 3-channel RGB, 3-channel scale, 4-channel rotation quaternion, 1-channel opacity, and 1-channel ray distance [S1]. The model uses a patch size of $p=8 \times 8$ for both patchifying and unpatchifying operations [S1]. The total output is $N \cdot HW$ Gaussians for $N$ input views, where $H$ and $W$ are image height and width, scaling with input resolution unlike fixed-resolution triplanes [S1]. This pixel-aligned scheme creates a shortcut for learning accurate per-Gaussian colors from input RGB images, unlike the non-pixel-aligned prediction of Triplane-LRM [S1].

## Background
The concept of direct pixel-wise regression for 3D Gaussian Splatting was introduced in GS-LRM (Large Reconstruction Model for 3D Gaussian Splatting) [S1]. The method was developed to address the limitations of fixed-resolution latent representations in prior Large Reconstruction Models (LRM) like Triplane-LRM, which struggled with high-frequency details and per-pixel color accuracy [S1]. By predicting Gaussians directly from image patches, GS-LRM allows the number of primitives to scale with input resolution, facilitating better handling of large-scale scene captures and high-frequency details [S1].

## Sources
[S1] GS-LRM: Large Reconstruction Model for 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-72670-5_1, fetched 2026-10-26
