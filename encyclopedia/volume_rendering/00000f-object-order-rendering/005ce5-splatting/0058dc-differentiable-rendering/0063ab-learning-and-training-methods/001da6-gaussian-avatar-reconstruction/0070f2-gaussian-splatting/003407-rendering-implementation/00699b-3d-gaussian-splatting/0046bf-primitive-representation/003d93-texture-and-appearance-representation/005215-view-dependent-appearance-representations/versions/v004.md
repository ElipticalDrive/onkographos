# View-Dependent Appearance Representations

Also called: Gaussian Appearance Encoding

## Overview
This family of methods extends the standard 3D Gaussian primitive with appearance attributes $\mathbf{a}_i$ to model view-dependent color $c_i(\mathbf{v}, \mathbf{a}_i)$, where $\mathbf{v}$ is the viewing direction [S1, unofficial]. Unlike static color storage, these representations decode color dynamically based on the viewing angle, enabling the modeling of specular highlights, anisotropy, and complex lighting effects [S1, unofficial]. The rendering pipeline retains front-to-back alpha-blending, but the color contribution of each Gaussian is computed via an appearance function that queries $\mathbf{a}_i$ using the viewing direction $\mathbf{v}$ [S1, unofficial]. This approach trades off storage bandwidth and decoding computational cost [S1, unofficial].

## Shared principles
All child methods extend the standard 3D Gaussian primitive $i$ with additional appearance attributes $\mathbf{a}_i$ beyond the geometric parameters and opacity $\alpha_i$ [G]. The rendering pipeline retains the front-to-back alpha-blending equation, but the color contribution $c_i$ of each Gaussian is no longer a static vector. Instead, $c_i$ is computed by evaluating an appearance function that queries $\mathbf{a}_i$ using the viewing direction $\mathbf{v}$ [S1, unofficial]. The final pixel color is calculated as:

$$
C(p) = \sum_{i \in \text{sorted}} c_i(\mathbf{v}, \mathbf{a}_i) \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j)
$$

where $c_i$ is the color contribution derived from the specific appearance representation [S1, unofficial]. This approach involves a trade-off between storage bandwidth and computational cost during decoding [S1, unofficial].

## Subtopics

### Fixed Basis Functions
Uses Spherical Harmonics (SH) for view-dependent color. Also supports Anisotropic Spherical Gaussians (ASG) for specular components and Spherical Gaussians (SG) for balancing quality and speed. Supports PBR parameters and indirect illumination. See [Fixed Basis Functions](8062-fixed-basis-functions.md).

### Neural Feature Vectors
Each point is associated with a compact Neural Feature Vector decoded using a small MLP. See [Neural Feature Vectors](8063-neural-feature-vectors.md).

### Triplane
Neural features are cached in explicit data structures. Position is projected onto axis-aligned feature planes, and final feature $f_t$ is obtained by concatenating three trilinear interpolated features. See [Triplane](8064-triplane.md).

### Hash Grid
Features are interpolated from a multi-resolution grid stored using instant-ngp. Supports unbounded scenes via contraction technique. See [Hash Grid](8065-hash-grid.md).

### Codebook Quantization
A discrete codebook of vector codes is learned during training. Rendering requires only a simple lookup into the codebook using stored indices, exploiting lower bandwidth and computational costs. See [Codebook Quantization](8066-codebook-quantization.md).

## Background
The concept of view-dependent appearance in Gaussian splatting was introduced as an enhancement step to boost the representation ability of 3D Gaussians [S1, unofficial]. Traditional Gaussian splatting utilized spherical harmonics (SH) to model view-dependent color [S1, unofficial]. Subsequent work expanded this to include Anisotropic Spherical Gaussians (ASG) for specular components and Spherical Gaussians (SG) for balancing image quality and rendering speed [S1, unofficial]. To achieve material and lighting decomposition, PBR parameters and indirect illumination were also considered [S1, unofficial]. Alternative approaches moved away from indexing neural feature vectors with point cloud IDs, caching features in explicit data structures like triplanes and hash grids [S1, unofficial]. Codebook quantization was motivated by the balance between compact storage and rendering efficiency, requiring only a simple lookup into the codebook using stored indices at rendering time [S1, unofficial].

## Sources
[S1] GauStudio: A Modular Framework for 3D Gaussian Splatting and Beyond, https://doi.org/10.48550/arXiv.2403.19632, fetched 2026-10-30 (unofficial)
