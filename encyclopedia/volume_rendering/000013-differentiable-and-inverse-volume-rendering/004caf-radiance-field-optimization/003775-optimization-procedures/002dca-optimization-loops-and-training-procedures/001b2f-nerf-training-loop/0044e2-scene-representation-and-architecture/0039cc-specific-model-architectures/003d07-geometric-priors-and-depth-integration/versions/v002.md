# Geometric Priors and Depth Integration

Also called: Geometric Constraints in NeRF, Depth-Guided Radiance Fields

## Overview
Geometric Priors and Depth Integration refers to a family of NeRF optimization strategies that incorporate explicit geometric constraints—such as Structure-from-Motion (SfM) correspondences, epipolar geometry, or monocular depth estimates—into the training loop to resolve geometric ambiguity in sparse-view or challenging scenarios [S1]. Unlike standard NeRF, which relies solely on implicit volume rendering and photometric consistency, this family enforces structural regularity through auxiliary loss terms, preventing local minima and improving geometry reconstruction fidelity [S1].

## Shared principles
The core mechanism involves augmenting the standard NeRF rendering loss with geometric regularization terms. These terms typically enforce consistency between the learned 3D geometry and external geometric signals. Common shared elements include:
1.  **Geometric Loss Terms:** Auxiliary losses such as $L_{geo}$ are added to the total loss function: $L_{total} = L_{render} + \lambda_{geo} L_{geo}$.
2.  **Depth Integration:** Methods often compute rendered depth $d_r$ from the density samples and compare it against prior depth estimates $d_{prior}$ or enforce depth ordering constraints.
3.  **Feature/Epipolar Constraints:** Some methods enforce that matched 2D features correspond to identical 3D coordinates or lie on epipolar lines, reducing the solution space for the radiance field.
4.  **Pyramidal/Coarse-to-Fine Training:** To handle large pixel motions and ensure convergence, many methods employ a pyramidal training strategy, starting from low-resolution images and progressively increasing resolution [S1].

## Subtopics

### [SfMNeRF](8664-sfmnerf.md)
SfMNeRF improves NeRF under sparse input conditions by integrating Structure-from-Motion (SfM) constraints and depth prediction into the training loop. It explicitly constrains 3D geometry using matched feature positions, epipolar geometry, and multi-view photometric consistency, reducing geometric ambiguity. Unlike standard NeRF, it adds auxiliary geometric loss terms to enforce that identical features share world coordinates and that corresponding points lie on epipolar lines. It employs a pyramidal training strategy to handle large pixel motions. See [SfMNeRF](8664-sfmnerf.md).

### [DASNeRF](8668-dasnerf.md)
DASNeRF is designed for sparse-view 3D reconstruction, integrating accurate monocular depth priors (specifically from DepthAnythingV2) to guide learning and employing a dynamic sampling mechanism to preserve fine details. It distinguishes itself by combining a five-dimensional per-layer input fusion strategy with robust relative depth regularization (Depth Ordering Fidelity and Depth Structural Consistency), rather than relying solely on SfM geometry or static depth constraints. See [DASNeRF](8668-dasnerf.md).

### GeoNeRF
GeoNeRF is a model comprising a geometric morphology reasoner and a renderer [S1]. The reasoner builds a cascaded cost volume for source view sequences, while the renderer infers geometric and appearance representations using a Transformer-based attention mechanism and these cost volumes, finally rendering fine images via volume rendering [S1]. This architecture allows for complex occlusion reasoning and extends NeRF to utilize RGB-D data [S1].

## Background
The integration of geometric priors into NeRF emerged to address the limitations of standard NeRF in sparse-view settings and its tendency to produce blurry or ambiguous geometry. Early works like SfMNeRF [S1] and DASNeRF [S1] demonstrated that explicit geometric constraints significantly improve reconstruction quality. GeoNeRF [S1] further explored Transformer-based geometric reasoning with cost volumes. These methods are primarily used in novel view synthesis and 3D scene reconstruction from sparse or uncalibrated inputs.

## Sources
[S1] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
