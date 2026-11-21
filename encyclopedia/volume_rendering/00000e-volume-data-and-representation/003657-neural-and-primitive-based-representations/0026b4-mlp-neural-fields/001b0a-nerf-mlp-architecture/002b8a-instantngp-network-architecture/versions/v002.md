# InstantNGP Network Architecture

Also called: Instant Neural Graphics Primitives Network, InstantNGP MLP, MonoSDF Network, Network Architecture and Implementation

## Overview
This article describes the MLP configuration and feature encoding strategy used in the InstantNGP framework, as implemented in Point-NeuS [S1, unofficial] and adapted for texture storage in HybridNeRF [S2]. It replaces the parent [NeRF MLP Architecture](910-nerf-mlp-architecture.md)'s raw positional encoding with multi-resolution feature storage.

## How it works
**Inputs and Outputs:**
The network takes 3D spatial coordinates $\mathbf{x}$ as input [S1, unofficial]. The outputs are latent features which are subsequently processed by specific head networks to produce signed distance values, density, or color [S1, unofficial].

**Data Structures:**
Multi-resolution features are stored in hash tables [S2]. The exact hash function, grid sizes, and feature vector dimensions are not given in the sources.

**Network Layout:**
The architecture consists of three distinct MLPs, all sharing the same InstantNGP input encoder [S1, unofficial]:
1. Base SDF Network: A 4-layer MLP with 256 hidden units [S1, unofficial].
2. Bias Network: A 2-layer MLP with a similar structure [S1, unofficial].
3. Color Network: A 2-layer MLP with 128 hidden units [S1, unofficial].

**Steps for Feature Encoding:**
InstantNGP encodes the input position information of the three networks using the same parameters as MonoSDF [S1, unofficial]. The specific encoding steps are not given in the sources.

**Parameters:**
* Hidden units are 256 for SDF and Bias networks, and 128 for the Color network [S1, unofficial].
* Layer counts are 4 for SDF and 2 for Bias and Color networks [S1, unofficial].
* The input point cloud is downsampled to 2 million points to demonstrate robustness to sparse data [S1, unofficial].
* Preprocessing generates depth maps and point clouds using TransMVSNet, taking approximately 1 second per image [S1, unofficial].

**Edge Cases and Pitfalls:**
Not given in the sources.

## Subtopics

### iNGP Texture Storage
A variant used in HybridNeRF [S2]. Instead of hash tables, it stores multi-resolution features as textures at render time to take advantage of memory locality and texture interpolation hardware [S2]. It stores presummed features $g'$ at resolution level $L$, where $g'(v) = \sum_{l=0}^{L} g(v, l)$ for each texel in $L$ [S2]. This allows fetching anti-aliased features by interpolating between two levels implied by pixel area $p(x)$ [S2]. The number of texture fetches per MLP evaluation is reduced to 8, from 24 (3+3×7) in the original approach, a 3× reduction [S2].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
The implementation demonstrates robustness with downsampled point clouds of 2 million points [S1, unofficial]. The texture storage variant reduces texture fetches by 3× [S2].

**Strengths and Limitations:**
Not given in the sources.

**When to Use:**
Not given in the sources.

**When Not to Use:**
Not given in the sources.

## Background
The architecture is utilized in Point-NeuS for surface reconstruction [S1, unofficial] and adapted in HybridNeRF for hybrid rendering [S2]. The MonoSDF variant uses a 4-layer 256-unit SDF network, a 2-layer 256-unit bias network, and a 2-layer 128-unit color network, all using InstantNGP encoding [S1, unofficial].

## Sources
[S1] Point-NeuS: Point-Guided Neural Implicit Surface Reconstruction by Volume Rendering, https://doi.org/10.48550/arXiv.2310.07997, fetched 2026-10-30 (unofficial)
[S2] HybridNeRF: Efficient Neural Rendering via Adaptive Volumetric Surfaces, https://doi.org/10.1109/cvpr52733.2024.01858, fetched 2026-10-26
