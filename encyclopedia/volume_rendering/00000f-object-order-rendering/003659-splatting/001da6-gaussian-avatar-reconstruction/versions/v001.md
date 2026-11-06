# Gaussian Avatar Reconstruction

Also called: Gaussian Avatars, 3DGS-Avatar, Relightable Gaussian Codec Avatars, NoPoSplat, GaussianShader, Animatable Gaussians, SplattingAvatar, Codec Avatar 3D, OmniRe

## Overview
Gaussian Avatar Reconstruction is a family of methods that reconstruct animatable, dynamic 3D human or head avatars from video by representing them as collections of 3D Gaussians [S1, unofficial][S2, unofficial]. It solves the problem of generating high-fidelity, real-time renderable avatars that can be driven by parametric body models (SMPL/FLAME) or pose features, serving as a dynamic alternative to static Neural Radiance Fields (NeRF) or traditional mesh-based animation [S1, unofficial][S2, unofficial]. It is distinguished from its sibling [Gaussian Splatting](946-gaussian-splatting.md) by its explicit dependency on a parametric structural prior to drive deformation, and from [Gaussian Avatars](1477-gaussian-avatars.md) which typically focus on head-only reconstruction using the FLAME model.

## Shared principles
The family shares the core mechanism of [Splatting](23-splatting.md), specifically the object-order rasterization of 3D Gaussian primitives [S1, unofficial][S4, unofficial]. The shared principles are:

1.  **Parametric Prior:** The geometry and deformation of the avatar are constrained by a parametric model (e.g., SMPL for full-body, FLAME for heads) [S1, unofficial][S2, unofficial]. The 3D Gaussians are initialized on or near this mesh surface [S2, unofficial].
2.  **Pose-Conditioned Deformation:** Gaussian parameters (position, rotation, scale) are functions of the parametric model's pose and shape parameters [S1, unofficial][S2, unofficial]. This allows the avatar to be animated by changing the input pose [S1, unofficial][S2, unofficial].
3.  **Differentiable Rendering:** The rendering process uses differentiable splatting to compute gradients from the rendered image to the Gaussian parameters and/or the deformation network weights [S1, unofficial][S2, unofficial].
4.  **Optimization:** The model is trained by minimizing photometric loss between rendered images and input video frames, often involving dynamic densification or MLP training for deformation [S1, unofficial][S2, unofficial].

## Subtopics

### [Gaussian Splatting](946-gaussian-splatting.md)
[See [Gaussian Splatting](946-gaussian-splatting.md)]

### [Gaussian Avatars](1477-gaussian-avatars.md)
[See [Gaussian Avatars](1477-gaussian-avatars.md)]

### [Gaussian Avatars](1477-gaussian-avatars.md)
[See [Gaussian Avatar](1479-gaussian-avatar.md)]

### [3DGS-Avatar](1480-3dgs-avatar.md)
[See [3DGS-Avatar](1480-3dgs-avatar.md)]

### [Relightable Gaussian Codec Avatars](1482-relightable-gaussian-codec-avatars.md)
[See [Relightable Gaussian Codec Avatars](1482-relightable-gaussian-codec-avatars.md)]

### [NoPoSplat](1484-noposplat.md)
[See [NoPoSplat](1484-noposplat.md)]

### GaussianShader
Enriches Gaussians with material parameters such as diffusiveness, specularity, roughness, and normals to handle specular and reflective objects [S1, unofficial].

### Animatable Gaussians
An avatar method using SMPL-X rig with explicit pose-conditioned offsets [S2, unofficial].

### SplattingAvatar
An avatar method using SMPL-X rig, embedded in mesh tangent space [S2, unofficial].

### Codec Avatar 3D
A production-grade GS-based head model for volumetric telepresence by Meta [S2, unofficial].

### OmniRe
A holistic scene reconstruction method including humans, vehicles, and static background [S2, unofficial].

## Background
The method originates from the application of 3D Gaussian Splatting to dynamic human reconstruction. Early works like HUGS introduced SMPL-driven body Gaussians [S2, unofficial]. Subsequent methods such as 3DGS-Avatar, GaussianAvatars, GoMAvatar, and AnimatableGaussians refined the approach using SMPL-X or FLAME models with various deformation strategies (MLPs, pose-conditioned offsets) [S2, unofficial]. Relightable variants like GoHA, GHG, and PSAvatar added BRDF modeling for lighting synthesis [S2, unofficial]. Meta's Codec Avatar 3D represents a production-grade implementation for telepresence [S2, unofficial]. The field has expanded to holistic scene reconstruction (OmniRe) and SLAM applications (SplaTAM, MonoGS) [S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
