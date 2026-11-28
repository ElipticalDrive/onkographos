# Transparency and Blending Models

Also called: Gaussian Splatting Transparency, Volume Rendering in 3DGS

## Overview
Transparency and Blending Models define how 3D Gaussian primitives are composited into a 2D image, addressing the fundamental problem of order-dependent alpha blending in splatting-based rendering. Standard 3D Gaussian Splatting uses approximate per-primitive sorting and 2D projection, which can introduce popping artifacts and inaccuracies in transmittance [2812-3d-gaussian-splatting-rasterization.md]. This family provides alternatives that either correct depth ordering for real-time performance, replace projection with analytic ray-space integration for physical accuracy, or eliminate sorting entirely via Order-Independent Transparency (OIT) [3294-hybrid-transparency.md], [4440-analytic-transmittance-3dgs.md], [7562-sparseoit-rasterizer.md].

## Shared principles
All methods in this family operate on a set of 3D Gaussians defined by mean $\boldsymbol{\mu}$, covariance $\boldsymbol{\Sigma}$ (or scales and orientation), opacity $o$, and color $\mathbf{c}$. The core challenge is computing the pixel color $C$ and accumulated alpha $\alpha$ given a camera ray. Standard splatting projects 3D Gaussians to 2D ellipses, computes a 2D alpha mask, and blends them in screen space. These models modify the blending stage: some refine the depth metric used for sorting, others integrate the 3D density function directly along the ray, and others use weighted averages to bypass sorting. All assume the Gaussians are sorted or processed in a manner that approximates front-to-back accumulation to ensure correct occlusion.

## Subtopics

### [Hybrid Transparency](3294-hybrid-transparency.md)
Combines depth-ordered alpha blending for the front $K$ fragments with order-independent accumulation for the tail. It solves popping artifacts by using per-pixel depth evaluation ($x_{view}$) for the core while avoiding the cost of full sorting. Choose when real-time performance is required but standard sorting causes visual artifacts.

### [Analytic Transmittance 3DGS](4440-analytic-transmittance-3dgs.md)
Replaces 2D splatting with analytic integration of the 3D Gaussian density along the camera ray. It computes exact transmittance without projecting to 2D, improving accuracy for opaque surfaces. Choose when view synthesis quality and physical accuracy are prioritized over the speed of 2D rasterization.

### [SparseOIT Rasterizer](7562-sparseoit-rasterizer.md)
Uses Order-Independent Transparency (OIT) to blend Gaussians without depth sorting. It computes a weighted average color and alpha, eliminating the sorting bottleneck. Choose when sorting is a performance bottleneck and approximate transparency is acceptable.

## Background
The concept stems from the original 3D Gaussian Splatting paper [Kerbl2023], which introduced alpha blending of projected 2D Gaussians. Subsequent work identified limitations in sorting stability and transmittance approximation. Hybrid Transparency addresses the "popping" issue in dynamic scenes [S1, S2]. Analytic Transmittance extends the volume rendering equation to 3D ray-space integration [S1]. SparseOIT adapts OIT techniques to the sparse nature of Gaussian splats [S1, unofficial].

## Sources
[S1] Hybrid Transparency, Analytic Transmittance, SparseOIT notes provided in prompt.
[S2] Hybrid Transparency notes provided in prompt.
