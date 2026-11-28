# LBS Network Variants

Also called: Backward LBS, Deterministic SMPL skinning, Learned forward LBS

## Overview
This family defines neural or deterministic methods for predicting Linear Blend Skinning (LBS) weights for 3D points in canonical space, enabling the deformation of articulated human avatars via learned bone affinities rather than fixed priors [S1]. It differs from standard LBS by replacing static weight maps with a differentiable function, allowing end-to-end optimization of skinning parameters alongside shape and appearance in differentiable rendering pipelines [S1]. Among its variants, the Learned Forward LBS achieves the highest quantitative performance (PSNR 31.6), outperforming Backward LBS (29.5) and Deterministic SMPL weights (30.5) [S1].

## Shared principles
All variants in this family map a query point $\hat{\mathbf{x}} \in \mathbb{R}^3$ in canonical space to a skinning weight vector $\mathbf{w} \in [0, 1]^{24}$, where each component $w_b$ represents the affinity of the point to bone $b$ [S1]. The deformation of a posed position $\bar{\mathbf{x}}$ is computed as a convex combination of bone transformations applied to the canonical point $\hat{\mathbf{x}}$ (treated as a homogeneous coordinate) [S1]:

$$ \bar{\mathbf{x}} = \left(\sum_{b=1}^{24} w_b \mathbf{B}_b\right) \hat{\mathbf{x}} $$

where $w_b$ is determined by the specific variant's method, and $\{\mathbf{B}_b\}_{b=1}^{24}$ are the 24 rigid bone transformation matrices derived from the SMPL model (23 local + 1 global) [S1]. The weights must satisfy $\sum w_b = 1$ and $w_b \ge 0$ [S1].

## Subtopics

### [Forward LBS Network](19756-forward-lbs-network.md)
A neural network (MLP) that predicts LBS weights for canonical points, enabling end-to-end optimization. It is chosen for its ability to avoid artifacts associated with fixed priors and achieves the highest PSNR (31.6).
- Backward LBS: A variant that conditions the LBS network on local body poses $\theta$. It canonicalizes query points using SMPL global translation and rotation before querying the network. It suffers from difficulties learning skinning weights for points far from the surface, resulting in artifacts under specific poses, and yields a lower PSNR of 29.5.
- Deterministic SMPL weights: A non-learned variant using nearest neighbor SMPL skinning weights. It causes discretized artifacts on the cloth surface and achieves a PSNR of 30.5.

## Background
The variants are evaluated in the context of animatable volume rendering of articulated human SDFs, specifically within the ARAH framework [S1]. The methods are tested on datasets such as CAPE [41] for meta-learning initialization and ZJU-MoCap [60] for per-subject optimization [S1]. The comparison highlights the trade-offs between learned differentiable approaches and deterministic or pose-conditioned baselines [S1].

## Sources
[S1] ARAH: Animatable Volume Rendering of Articulated Human SDFs, https://doi.org/10.1007/978-3-031-19824-3_1, fetched 2026-10-26
