# SMPL-based rigid/non-rigid deformation

Also called: Parametric Gaussian Avatars, SMPL-Gaussian Deformation

## Overview
This family reconstructs animatable 3D human avatars by driving 3D Gaussian primitives with the SMPL parametric model [S1, unofficial][S2, unofficial]. It solves the problem of rendering dynamic, pose-dependent cloth deformation and fine geometric details that standard static splatting cannot capture [S1]. It differs from static [Gaussian Splatting](946-gaussian-splatting.md) by predicting Gaussian parameters as functions of the model's pose, and from sibling methods like [Gaussian Avatar](1479-gaussian-avatar.md) by using explicit rigid/non-rigid transformation combinations rather than purely feed-forward networks for deformation [S1, unofficial][S2, unofficial].

## Shared principles
The family builds on [Splatting](23-splatting.md) by treating volumetric data as a set of parametric primitives that are transformed and rasterized [S1, unofficial][S2, unofficial].

1.  **Parametric Prior:** The geometry and deformation of the avatar are constrained by a parametric model (e.g., SMPL for full-body, FLAME for heads) [S2, unofficial].
2.  **Pose-Conditioned Deformation:** Gaussian parameters (position, rotation, scale) are functions of the parametric model's pose and shape parameters [S2, unofficial].
3.  **Differentiable Rendering:** The rendering process uses differentiable splatting to compute gradients from the rendered image to the Gaussian parameters and/or the deformation network weights [S1, unofficial][S2, unofficial].
4.  **Optimization:** The model is trained by minimizing photometric loss between rendered images and input video frames [S2, unofficial].

## Subtopics

### [Gaussian Avatar](1479-gaussian-avatar.md)
Reconstructs animatable full-body human avatars by leveraging the SMPL parametric model as a geometric prior. It differs from static [Gaussian Splatting](946-gaussian-splatting.md) by predicting Gaussian parameters as functions of the model's pose, and from sibling methods like 3DGS-Avatar by using a feed-forward network for pose-conditioned deformation rather than explicit rigid/non-rigid transformation combinations.

### [3DGS-Avatar](1480-3dgs-avatar.md)
Reconstructs photorealistic, animatable full-body human avatars by combining 3D Gaussian primitives with an underlying SMPL model. It addresses the challenge of rendering pose-dependent cloth deformation and fine geometric details that standard spherical harmonics (SH) often blur. It distinguishes itself from sibling methods like [Gaussian Avatar](1479-gaussian-avatar.md) by employing a shallow Multi-Layer Perceptron (MLP) for color modeling and explicit geometry priors for deformation regularization.

### [Animatable Gaussians](6157-animatable-gaussians.md)
Reconstructs animatable 3D human avatars from monocular video by driving explicit 3D Gaussians with an SMPL-X rig. It differs from sibling methods like [3DGS-Avatar](1480-3dgs-avatar.md), which uses a per-Gaussian deformation MLP, by employing explicit pose-conditioned offsets to determine Gaussian positions. This approach provides a direct mapping between the parametric model's pose and the Gaussian geometry, avoiding the need for learned implicit deformation fields for position.

## Background
The method originates from the integration of parametric human models (SMPL) with 3D Gaussian Splatting to enable real-time rendering of dynamic humans. Early work focused on static avatars, but the shift to pose-conditioned deformation allowed for animatable characters from monocular or multi-view video [S1, unofficial][S2, unofficial]. It is primarily used in virtual try-on, digital humans, and video compression where explicit geometry is required [S1, unofficial][S2, unofficial].

## Sources
[S1] Gaussian Avatar, https://github.com/bertjiazheng/GaussianAvatar, 2023-10-15
[S2] Gaussian Splatting for Real-time Radiance Field Rendering, https://github.com/graphdeco-inria/gaussian-splatting, 2023-09-01
[S3] 3DGS-Avatar, https://github.com/Helios-3D/3DGS-Avatar, 2023-11-20
