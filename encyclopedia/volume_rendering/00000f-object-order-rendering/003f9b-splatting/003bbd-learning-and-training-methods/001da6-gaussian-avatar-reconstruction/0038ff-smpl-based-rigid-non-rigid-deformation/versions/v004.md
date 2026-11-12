# SMPL-based rigid/non-rigid deformation

Also called: SMPL-based deformation, Hybrid deformation strategy

## Overview
This family of methods reconstructs animatable human avatars by anchoring 3D Gaussians to the SMPL parametric body model in a canonical rest pose [S1]. It solves the problem of dynamic human reconstruction by combining rigid skeletal transformations (via Linear Blend Skinning) with non-rigid deformation fields to model secondary effects like muscle movement and cloth dynamics [S1]. It is distinguished from siblings like [Gaussian Splatting](946-gaussian-splatting.md) by its explicit reliance on a parametric geometric prior to ensure structural integrity during animation, and from [Gaussian Avatars](1477-gaussian-avatars.md) by the specific categorization of deformation into rigid LBS and non-rigid residual components [S1].

## Shared principles
All methods in this family rely on differentiable rendering to compute gradients from the rendered image to the Gaussian parameters (position, covariance, opacity, spherical harmonics) and/or network weights [S1, 6136-instantsplat.md, 6464-pixelsplat.md]. The rendering process remains differentiable with respect to the scene representation [S1, 6136-instantsplat.md, 6464-pixelsplat.md]. Loss functions typically combine photometric reconstruction error, such as $\mathcal{L}_{L1}$ or $\mathcal{L}_{MSE}$, with perceptual or structural metrics like $\mathcal{L}_{MS-SSIM}$ or $\mathcal{L}_{LPIPS}$ [6136-instantsplat.md][6464-pixelsplat.md].

A core principle is the anchoring of Gaussians to a geometric prior. Most methods initialize Gaussians on the surface of a parametric mesh (e.g., SMPL for full body, FLAME for heads) in a canonical rest pose [S1, 1477-gaussian-avatars.md, 1482-relightable-gaussian-codec-avatars.md]. During training, these Gaussians are deformed to match the target pose using rigid transformations (e.g., Linear Blend Skinning) and/or non-rigid deformation fields [S1, 8385-smpl-based-rigid-non-rigid-deformation.md]. This ensures that the avatar can be animated to novel poses while maintaining structural integrity [S1].

The deformation strategy is hybrid:
1.  **Rigid Transformation:** Linear Blend Skinning (LBS) is applied to the canonical Gaussian positions $\mathbf{p}$ using SMPL joint rotations $\theta$ and skinning weights $w_j$ to produce $\mathbf{p}_{rigid}$ [S1, 1480-3dgs-avatar.md, 6157-animatable-gaussians.md].
    $$ \mathbf{p}_{rigid} = \text{LBS}(\mathbf{p}, \theta) = \sum_{j=1}^{N_{bones}} w_j(\mathbf{p}, \theta) \mathbf{T}_j(\theta) \mathbf{p} $$
2.  **Non-Rigid Deformation:** A residual offset $\Delta \mathbf{p}$ is predicted by a deformation network conditioned on pose, time, or spatial coordinates, which is added to the rigidly transformed position [S1, 1480-3dgs-avatar.md, 6157-animatable-gaussians.md, 11653-uv-based-motion-parameterization.md].
    $$ \mathbf{p}' = \mathbf{p}_{rigid} + \Delta \mathbf{p} $$

The covariance matrix $\Sigma$ is updated to reflect the deformation, typically using the Jacobian $J$ of the deformation map with respect to the canonical position [G]:
$$ \Sigma' = J \Sigma J^T + \sigma^2 I $$

## Subtopics

### [Gaussian Avatar](1479-gaussian-avatar.md)
Gaussian Avatar is a method for reconstructing animatable full-body human avatars by leveraging the SMPL parametric model as a geometric prior. It solves the problem of dynamic human reconstruction by predicting Gaussian parameters (position, rotation, scale, color) from pose features via a feed-forward network, enabling real-time novel view synthesis and animation. It is distinguished from siblings like [3DGS-Avatar](1480-3dgs-avatar.md) by its specific reliance on pose features driving a feed-forward network for parameter prediction, whereas 3DGS-Avatar explicitly combines non-rigid and rigid transformations of the SMPL model.

### [3DGS-Avatar](1480-3dgs-avatar.md)
3DGS-Avatar reconstructs full-body digital humans by combining 3D Gaussians with rigid and non-rigid transformations derived from an underlying SMPL model. It distinguishes itself from sibling methods by introducing a shallow Multi-Layer Perceptron (MLP) to model Gaussian color, replacing spherical harmonics, and by regularizing the deformation with geometry priors to handle pose-dependent cloth deformation effectively.

### [Animatable Gaussians](6157-animatable-gaussians.md)
Animatable Gaussians is a method for reconstructing animatable 3D human avatars using explicit pose-conditioned offsets within an SMPL-X rig. It solves the problem of generating dynamic, high-fidelity human representations from static or video data by decoupling rigid skeletal motion from non-rigid surface deformation. It is distinguished from siblings like [Gaussian Avatar](1479-gaussian-avatar.md) and [3DGS-Avatar](1480-3dgs-avatar.md) by its use of explicit, analytically defined pose-conditioned offset fields rather than implicit MLPs or UV-based parameterizations.

### [UV-based motion parameterization](11653-uv-based-motion-parameterization.md)
UV-based motion parameterization is a technique for modeling dynamic human deformation by reparameterizing 3D spatial coordinates into 2D surface-level UV coordinates combined with time. It solves the problem of high-dimensional motion field learning by exploiting the prior that human motion occurs primarily at the surface level, reducing the input dimensionality from 4D space-time to 3D surface-time. This approach is significantly more memory efficient and converges faster than methods using raw 3D coordinates.

#### Full-body Avatar Modeling
**Definition**
A method that anchors 3D Gaussians in a canonical space and deforms them via parametric body models or cage-based rigging to model dynamic motions, using a hybrid deformation strategy [S1].

**Steps**
The hybrid deformation strategy uses linear blend skinning to handle rigid skeletal transformations such as joint rotations [S1]. The hybrid deformation strategy uses pose-conditioned deformation fields to account for secondary non-rigid effects like muscle jiggles [S1].

## Background
The approach originates from the application of 3D Gaussian Splatting to human avatar reconstruction, specifically for full-body modeling [S1]. Current methods typically anchor 3D Gaussians in a canonical space and deform them via parametric body models or cage-based rigging to model dynamic motions [S1]. These approaches adopt a hybrid deformation strategy: linear blend skinning handles rigid skeletal transformations such as joint rotations, while pose-conditioned deformation fields account for secondary non-rigid effects like muscle jiggles [S1]. This family is part of the broader advancement of 3D GS in redefining social, industrial, and creative landscapes through VR and dynamic content creation [S1].

## Sources
[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
