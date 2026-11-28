# SMPL-based rigid/non-rigid deformation

Also called: SMPL-driven Gaussian deformation, SMPL-conditioned Gaussians

## Overview
This family reconstructs animatable human avatars by initializing 3D Gaussian primitives on a parametric body model (SMPL or SMPL-X) and conditioning their deformation on pose parameters [S1, unofficial][S2, unofficial]. It differs from static [Gaussian Splatting](946-gaussian-splatting.md) by introducing pose-dependent geometry updates, and from head-focused methods like [Gaussian Avatars](1477-gaussian-avatars.md) which use the FLAME model for facial reconstruction [S3, unofficial]. The approach solves the problem of high-fidelity, real-time rendering of dynamic full-body humans from video by combining differentiable splatting with a parametric prior [S1, unofficial][S2, unofficial].

## Shared principles
The family shares the core mechanism of [Splatting](23-splatting.md), specifically the object-order rasterization of 3D Gaussian primitives [S1, unofficial][S4, unofficial]. The shared principles are:

1.  **Parametric Prior:** The geometry and deformation of the avatar are constrained by a parametric model (e.g., SMPL for full-body, FLAME for heads) [S1, unofficial][S2, unofficial]. The 3D Gaussians are initialized on or near this mesh surface [S2, unofficial].
2.  **Pose-Conditioned Deformation:** Gaussian parameters (position, rotation, scale) are functions of the parametric model's pose and shape parameters [S1, unofficial][S2, unofficial]. This allows the avatar to be animated by changing the input pose [S1, unofficial][S2, unofficial].
3.  **Differentiable Rendering:** The rendering process uses differentiable splatting to compute gradients from the rendered image to the Gaussian parameters and/or the deformation network weights [S1, unofficial][S2, unofficial].
4.  **Optimization:** The model is trained by minimizing photometric loss between rendered images and input video frames, often involving dynamic densification or MLP training for deformation [S1, unofficial][S2, unofficial].

## Subtopics

### [Gaussian Avatar](1479-gaussian-avatar.md)
[Gaussian Avatar](1479-gaussian-avatar.md) is a method for reconstructing animatable full-body human avatars by leveraging the SMPL parametric model as a geometric prior. It differs from static [Gaussian Splatting](946-gaussian-splatting.md) by predicting Gaussian parameters (position, rotation, scale) as functions of the model's pose, and from [Gaussian Avatars](1477-gaussian-avatars.md) which typically focuses on head/face using FLAME. It solves the problem of high-fidelity, real-time rendering of dynamic humans from video by combining differentiable splatting with pose-conditioned deformation.

### [3DGS-Avatar](1480-3dgs-avatar.md)
[3DGS-Avatar](1480-3dgs-avatar.md) is a full-body avatar reconstruction method that combines 3D Gaussian primitives with an underlying SMPL parametric model. It solves the problem of animating high-fidelity human avatars by applying both non-rigid and rigid transformations to Gaussians based on the SMPL pose. It is distinguished from its siblings by its use of the SMPL-X rig and per-Gaussian deformation MLPs, whereas Gaussian Avatars uses FLAME for heads and AnimatableGaussians uses explicit pose-conditioned offsets.

### [Animatable Gaussians](6157-animatable-gaussians.md)
[Animatable Gaussians](6157-animatable-gaussians.md) is an avatar method using SMPL-X rig with explicit pose-conditioned offsets.

### SplattingAvatar
[SplattingAvatar](1479-gaussian-avatar.md) is an avatar method using SMPL-X rig, embedded in mesh tangent space.

## Background
The approach originates from the integration of parametric human models (SMPL [S1, unofficial]) with 3D Gaussian Splatting [S1, unofficial]. Early works like HUGS and 3DGS-Avatar established the SMPL-driven body GS framework [S1, unofficial]. Subsequent developments introduced SMPL-X rigs [S1, unofficial], per-Gaussian deformation MLPs [S1, unofficial], and mesh-coupled monocular video reconstruction (GoMAvatar) [S1, unofficial]. The family distinguishes itself from head-driven methods like GoHA/GoHA (FLAME) [S1, unofficial] and strand-based methods like GaussianHair [S1, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
