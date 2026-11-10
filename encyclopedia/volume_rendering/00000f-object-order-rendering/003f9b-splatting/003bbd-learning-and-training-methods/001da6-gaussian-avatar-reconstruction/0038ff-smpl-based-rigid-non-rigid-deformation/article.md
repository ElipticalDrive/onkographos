# SMPL-based rigid/non-rigid deformation

Also called: SMPL-based Gaussian Deformation, Parametric Gaussian Avatar Deformation

## Overview
This family of methods reconstructs animatable 3D human avatars by anchoring 3D Gaussian primitives to a parametric body model, typically SMPL or SMPL-X, to handle dynamic non-rigid deformation [S1]. It solves the problem of generalizing to novel poses and handling complex clothing interactions by combining rigid skeletal transformations with pose-conditioned non-rigid deformation fields, distinguishing itself from generic Gaussian Splatting [946-gaussian-splatting.md] which lacks explicit geometric priors for animation.

## Shared principles
All methods in this family anchor 3D Gaussians in a canonical space (the rest pose of the parametric model) and deform them to match the target pose [S1]. The deformation strategy is hybrid:
1.  **Rigid Transformation:** Linear Blend Skinning (LBS) derived from the parametric model (SMPL/SMPL-X) handles the primary rigid skeletal transformations, such as joint rotations and global translation [S1]. For a Gaussian at canonical position $\mathbf{p}$, the rigid transformation is $T_{rigid}(\mathbf{p}, \theta)$, where $\theta$ represents the pose parameters.
2.  **Non-Rigid Deformation:** A pose-conditioned deformation field or network predicts residual offsets $\Delta \mathbf{p}$ to account for secondary non-rigid effects, such as muscle jiggles, cloth dynamics, or volume preservation, which LBS alone cannot capture [S1]. The final deformed position is $\mathbf{p}' = T_{rigid}(\mathbf{p}, \theta) + \Delta \mathbf{p}(\theta)$.
3.  **Initialization:** Gaussians are initialized on the surface of the parametric mesh in the canonical rest pose [S1, 1479-gaussian-avatar.md, 1480-3dgs-avatar.md, 6157-animatable-gaussians.md].
4.  **Differentiable Rendering:** The deformed Gaussians are rendered using differentiable splatting, allowing gradients to flow from the photometric loss to the deformation network parameters and Gaussian attributes [1479-gaussian-avatar.md, 1480-3dgs-avatar.md, 6157-animatable-gaussians.md].

## Subtopics

### [Gaussian Avatar](1479-gaussian-avatar.md)
[1479-gaussian-avatar.md] predicts Gaussian parameters (position, rotation, scale, color) directly from a pose feature vector via a feed-forward network, relying on the SMPL prior for geometric consistency [1479-gaussian-avatar.md].

### [3DGS-Avatar](1480-3dgs-avatar.md)
[1480-3dgs-avatar.md] combines rigid SMPL transformations with per-Gaussian non-rigid offsets predicted by a shallow MLP, replacing spherical harmonics with a color MLP to better handle pose-dependent cloth deformation [1480-3dgs-avatar.md].

### [Animatable Gaussians](6157-animatable-gaussians.md)
[6157-animatable-gaussians.md] uses the SMPL-X model and employs explicit pose-conditioned offsets to transform Gaussian primitives, differing from methods that use per-Gaussian deformation MLPs by applying offsets to the base SMPL-X skinning result [6157-animatable-gaussians.md].

### [UV-based motion parameterization](11653-uv-based-motion-parameterization.md)
[11653-uv-based-motion-parameterization.md] reparameterizes 4D space-time queries into a 3D surface-time domain using 2D UV coordinates and time, reducing the computational complexity of the residual deformation network by exploiting the prior that human motion is surface-local [11653-uv-based-motion-parameterization.md].

### Full-body Avatar Modeling
A method that anchors 3D Gaussians in a canonical space and deforms them via parametric body models or cage-based rigging to model dynamic motions, using a hybrid deformation strategy [S1]. The hybrid deformation strategy uses linear blend skinning to handle rigid skeletal transformations such as joint rotations [S1]. The hybrid deformation strategy uses pose-conditioned deformation fields to account for secondary non-rigid effects like muscle jiggles [S1].

## Background
The approach emerged as 3D Gaussian Splatting (3DGS) advanced along two directions: full-body modeling and head-centric modeling [S1]. For full-body avatars, current methods typically anchor 3D Gaussians in a canonical space and deform them via parametric body models or cage-based rigging to model dynamic motions [S1]. This direction addresses the limitations of static 3DGS by enabling animatable avatars, leveraging the speed advantage and editability of GS [S1].

## Sources
[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
