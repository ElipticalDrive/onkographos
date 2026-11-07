# TensoRF
Also called: Tensorf
## Overview
TensoRF models the complete radiance field as a 4D tensor and decomposes it into compressed, low-rank tensor components, replacing the sole reliance on fully connected MLPs found in standard NeRF [S1, unofficial]. This approach aims for computational efficiency, offering faster training times and lower space complexity compared to standard MLP-based NeRFs [S1, unofficial].
## Shared principles
TensoRF replaces the single global MLP with a tensorial representation of the scene. The radiance field is treated as a 4D tensor $\mathcal{T} \in \mathbb{R}^{H \times W \times D \times C}$ (or similar dimensionality depending on discretization), which is factorized into a set of low-rank components. This decomposition allows for efficient storage and computation of density $\sigma$ and color $\mathbf{c}$ without the quadratic or cubic parameter growth associated with dense MLPs in 3D space. The core mechanism relies on tensor decomposition techniques (such as CP or Tucker decomposition) to reconstruct the field values from these compact factors.
## Subtopics
## Background
TensoRF was introduced by Anpei Chen, Zexiang Xu, Andreas Geiger, Jingyi Yu, and Hao Su in the paper "Tensorf: Tensorial radiance fields" presented at the European Conference on Computer Vision (ECCV) in 2022 [S1, unofficial]. The method was developed to address the high memory and computational costs of traditional NeRF architectures.
## Sources
[S1] Low-Cost Neural Radiance Fields, https://doi.org/10.48550/arxiv.2605.09312, fetched 2026-10-26 (unofficial)
