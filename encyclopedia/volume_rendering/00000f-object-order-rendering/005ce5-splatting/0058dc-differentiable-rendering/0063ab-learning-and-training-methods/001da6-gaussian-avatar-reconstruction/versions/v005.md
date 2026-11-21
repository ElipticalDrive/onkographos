# Gaussian Avatar Reconstruction

Also called: Animatable Gaussian Avatars, Dynamic 3D Gaussian Splatting for Humans

## Overview
Gaussian Avatar Reconstruction refers to a family of methods that reconstruct animatable, photorealistic 3D human or head avatars by representing them as collections of 3D Gaussians [S1, unofficial]. Unlike static scene reconstruction, these methods must handle non-rigid deformation, ensuring that the Gaussian primitives move and deform consistently with the underlying subject's pose and expression [S1, unofficial]. The family is distinguished from static Gaussian Splatting [946-gaussian-splatting.md] by its integration of parametric body models (e.g., SMPL, FLAME) or deformation fields to constrain and drive the Gaussian positions, and from neural radiance field (NeRF) avatars by its explicit, rasterizable representation that enables real-time rendering speeds [S1, unofficial].

## Shared principles
All methods in this family rely on differentiable rendering to compute gradients from the rendered image to the Gaussian parameters (position, covariance, opacity, spherical harmonics) and/or network weights [S1, 6136-instantsplat.md, 6464-pixelsplat.md, unofficial]. The rendering process remains differentiable with respect to the scene representation [S1, 6136-instantsplat.md, 6464-pixelsplat.md, unofficial]. Loss functions typically combine photometric reconstruction error, such as $\mathcal{L}_{L1}$ or $\mathcal{L}_{MSE}$, with perceptual or structural metrics like $\mathcal{L}_{MS-SSIM}$ or $\mathcal{L}_{LPIPS}$ [6136-instantsplat.md][6464-pixelsplat.md].

A core principle is the anchoring of Gaussians to a geometric prior. Most methods initialize Gaussians on the surface of a parametric mesh (e.g., SMPL for full body, FLAME for heads) in a canonical rest pose [S1, 1477-gaussian-avatars.md, 1482-relightable-gaussian-codec-avatars.md, unofficial]. During training, these Gaussians are deformed to match the target pose using rigid transformations (e.g., Linear Blend Skinning) and/or non-rigid deformation fields [S1, 8385-smpl-based-rigid-non-rigid-deformation.md, unofficial]. This ensures that the avatar can be animated to novel poses while maintaining structural integrity [S1, unofficial].

## Subtopics

### [Gaussian Splatting](946-gaussian-splatting.md)
The foundational static rendering method. Not an avatar method itself, but the rendering engine used by all avatar variants.

### [GaussianShader](1470-gaussianshader.md)
Enriches Gaussians with physical material parameters (diffusiveness, specularity, roughness, normals) to enable specular and reflective rendering. Distinguished by explicit material properties rather than view-dependent color alone.

### [SpotlessSplats](1476-spotlesssplats.md)
Enhances robustness against transient distractors (e.g., moving people, lighting changes) in casual video captures by detecting outliers in a pre-trained feature space. Builds on Gaussian Avatar Reconstruction by adding feature-based outlier detection.

### [Gaussian Avatars](1477-gaussian-avatars.md)
Reconstructs photorealistic, animatable head avatars by associating 3D Gaussians with FLAME mesh triangles. Distinguished by explicit triangle-to-Gaussian mapping rather than MLP-based position prediction, and focuses on the head/bust region.

### [Relightable Gaussian Codec Avatars](1482-relightable-gaussian-codec-avatars.md)
Creates photorealistic, animatable 3D head avatars that support dynamic relighting under diverse lighting conditions. Builds on Gaussian Avatar Reconstruction by integrating a relightable appearance model with a FLAME-based parametric rig.

### [NoPoSplat](1484-noposplat.md)
A 3D Gaussian Splatting method for novel view synthesis from sparse or unposed input images. Eliminates dependency on known camera poses by establishing a canonical coordinate system anchored to a single input view and using intrinsics token embeddings.

### [SMPL-based rigid/non-rigid deformation](8385-smpl-based-rigid-non-rigid-deformation.md)
Reconstructs animatable 3D human avatars by anchoring Gaussians to SMPL/SMPL-X models. Solves generalization to novel poses by combining rigid skeletal transformations with pose-conditioned non-rigid deformation fields.

### Codec Avatar 3D
- [S1, unofficial] A production-grade GS-based head model for volumetric telepresence by Meta.

### OmniRe
- [S1, unofficial] A holistic scene reconstruction method including humans, vehicles, and static background.

### SplatArmor
- [S2] Leverage two different MLPs to predict large motions built upon the SMPL and canonical space.
- [S2] Allow pose-dependent effects by the proposed SE(3) fields.

## Background
The field emerged from the success of 3D Gaussian Splatting [946-gaussian-splatting.md] for static scene rendering [S1, unofficial]. Early attempts to animate Gaussians relied on neural deformation fields or tetrahedral cages [S2]. The introduction of parametric model integration, such as Gaussian Avatars using FLAME [S1, 1477-gaussian-avatars.md, unofficial] and SMPL-based methods [S1, 8385-smpl-based-rigid-non-rigid-deformation.md, unofficial], provided a robust geometric prior for non-rigid deformation. Recent developments include relightable avatars [S2, 1482-relightable-gaussian-codec-avatars.md] and robust training against distractors [S1, 1476-spotlesssplats.md, unofficial]. Production-grade implementations like Meta's Codec Avatar 3D [S1, unofficial] and holistic scene reconstruction methods like OmniRe [S1, unofficial] indicate industry adoption for telepresence and simulation [S1, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
