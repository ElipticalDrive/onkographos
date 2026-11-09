# Geometric Priors and Depth Integration

Also called: Geometric Regularization in NeRF, Depth-Guided Radiance Fields

## Overview
This family of methods enhances the geometric accuracy and structural consistency of Neural Radiance Fields (NeRF) by integrating external geometric constraints or priors into the optimization process. Standard NeRF relies solely on photometric consistency, which often leads to ambiguous geometry, especially under sparse input conditions or in textureless regions. By incorporating priors such as Structure-from-Motion (SfM) features, monocular depth estimates, or explicit geometric regularizers, these methods guide the implicit surface reconstruction to be more faithful to the real-world 3D structure. They differ from sibling methods like [AdaNeRF](6195-adanerf.md) (which adapts sampling) or [SinNeRF](6199-sinnerf.md) (which uses semantic regularization) by focusing specifically on enforcing geometric fidelity through depth ordering, epipolar constraints, or explicit surface representations.

## Shared principles
The core principle is the augmentation of the standard NeRF photometric loss ($L_{ren}$) with geometric regularization terms ($L_{geo}$). The total loss becomes:
$$
L_{total} = L_{ren} + \lambda L_{geo}
$$
where $\lambda$ is a weighting hyperparameter. The geometric term $L_{geo}$ varies by method but generally aims to:
1.  **Enforce Depth Ordering:** Ensure that the rendered depth respects the relative depth relationships provided by a prior (e.g., monocular depth estimator) [S1].
2.  **Constrain 3D Structure:** Use multi-view geometric constraints (epipolar geometry, feature matches) to anchor the 3D coordinates of the radiance field [S1].
3.  **Regularize Surface Smoothness:** Apply smoothness constraints on the depth map or density field to reduce noise and artifacts [S1].

The input to the MLP may remain standard (position $\mathbf{x}$, direction $\mathbf{d}$) or be modified to include geometric features (e.g., fused 5D vectors) [S1].

## Subtopics

### [SfMNeRF](8664-sfmnerf.md)
Integrates Structure-from-Motion (SfM) constraints, including matched SIFT features and epipolar geometry, into the training loop. It distinguishes itself by explicitly constraining 3D structure using sparse feature matches and dense photometric warping, addressing local minima in sparse views via a pyramidal strategy. Choose when input images are sparse and high-precision 3D structure is required.

### [DASNeRF](8668-dasnerf.md)
Combines accurate monocular depth priors (DepthAnythingV2) with a dynamic sampling mechanism. It uses specialized losses for Depth Ordering Fidelity and Depth Structural Consistency. Choose when dense depth priors are available and high-frequency geometric details need preservation without multi-view feature matching.

### GeoNeRF
A model consisting of a geometric morphology reasoner and a renderer. The reasoner builds a cascaded cost volume for source view sequences, and the renderer infers geometric and appearance representations using a Transformer-based attention mechanism and cascaded cost volumes, then renders fine images using volume rendering. It allows for complex occlusion reasoning and extends to RGB-D images.

## Background
The integration of geometric priors into NeRF emerged to address the "floaters" and ambiguous geometry issues inherent in implicit radiance fields trained only on color loss. Early works like [SfMNeRF](8664-sfmnerf.md) (2022) introduced SfM constraints to handle sparse inputs [S1]. Subsequent methods like [DASNeRF](8668-dasnerf.md) leveraged advances in monocular depth estimation (e.g., DepthAnythingV2) to provide dense geometric guidance [S1]. GeoNeRF (2022) proposed a Transformer-based approach with cascaded cost volumes for geometric reasoning [S1]. These methods are widely used in 3D reconstruction from sparse views, AR/VR content creation, and scenarios where explicit geometry is as important as photorealistic rendering.

## Sources
[S1] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
