# Gaussian Avatar Reconstruction

Also called: Gaussian Avatars (family), Parametric Gaussian Avatars

## Overview
Gaussian Avatar Reconstruction methods represent dynamic human subjects (typically heads or full bodies) as a set of deformable 3D Gaussian primitives driven by parametric models like FLAME or SMPL [S1, unofficial][S2, unofficial]. Unlike static [Gaussian Splatting](946-gaussian-splatting.md) which assumes a rigid scene, this family solves the problem of rendering animatable, high-fidelity avatars from video by constraining Gaussian geometry to the parametric mesh surface and optimizing pose-dependent deformations [S1, unofficial][S2, unofficial]. It differs from sibling methods like [Pose-free and Feed-forward Training](6500-pose-free-and-feed-forward-training.md) by explicitly relying on known or estimated camera poses and parametric priors for deformation, rather than learning implicit mappings from unposed images [S1, unofficial].

## Shared principles
The methods in this family share the fundamental mechanism of treating volumetric data as a set of parametric primitives that are transformed and rasterized [S1, unofficial][S2, unofficial]. All variants rely on differentiable rendering to compute gradients from the rendered image to the Gaussian parameters (position, covariance, opacity, spherical harmonics) and/or network weights [S1, 6136-instantsplat.md, 6464-pixelsplat.md, unofficial]. The rendering process remains differentiable with respect to the scene representation [S1, 6136-instantsplat.md, 6464-pixelsplat.md, unofficial]. Loss functions typically combine photometric reconstruction error, such as $\mathcal{L}_{L1}$ or $\mathcal{L}_{MSE}$, with perceptual or structural metrics like $\mathcal{L}_{MS-SSIM}$ or $\mathcal{L}_{LPIPS}$ [6136-instantsplat.md][6464-pixelsplat.md].

Specific to this family:
*   **Parametric Constraint:** Gaussian positions and orientations are initialized and constrained to the surface of a parametric mesh (e.g., FLAME for heads, SMPL for bodies) [S1, unofficial][S2, unofficial].
*   **Deformation:** Gaussian parameters are functions of the parametric model's pose and shape parameters [S2, unofficial]. This allows the avatar to be animated by changing the pose/shape inputs.
*   **Initialization:** Gaussians are typically initialized per triangle or vertex of the parametric mesh [S1, unofficial][S2, unofficial].

## Subtopics

### [Gaussian Splatting](946-gaussian-splatting.md)
3D Gaussian Splatting (3DGS) is the baseline architecture for explicit, learnable Gaussian primitives optimized directly from images. It solves the computational bottleneck of implicit neural radiance fields (NeRFs) by replacing ray marching with a tile-based differentiable rasterization pipeline, enabling real-time rendering speeds (≥30 fps at 1080p) while maintaining photorealistic quality. Unlike its siblings [SpotlessSplats](1476-spotlesssplats.md) and [Gaussian Avatars](1477-gaussian-avatars.md) which focus on specific deformation or optimization constraints, 3DGS establishes the baseline architecture for static scene representation.

### [SpotlessSplats](1476-spotlesssplats.md)
SpotlessSplats is a robust Gaussian Avatar Reconstruction method designed to improve reconstruction quality in casual captures by ignoring transient distractors. It enhances robustness by detecting outliers in a richer, pre-trained feature space. Unlike standard Gaussian Splatting which may be sensitive to noise, SpotlessSplats builds on [Gaussian Avatar Reconstruction](1575-gaussian-avatar-reconstruction.md) by integrating a feature-based outlier detection mechanism to filter out transient effects and occluders.

### [Gaussian Avatars](1477-gaussian-avatars.md)
Gaussian Avatars is a head-driven, riggable avatar reconstruction method that uses the FLAME parametric model to constrain 3D Gaussians to the surface of a bust mesh. It differs from full-body approaches like [Gaussian Avatar](1575-gaussian-avatar-reconstruction.md) by focusing on high-fidelity facial details and enabling relighting through a light rig.

### [Relightable Gaussian Codec Avatars](1482-relightable-gaussian-codec-avatars.md)
Relightable Gaussian Codec Avatars is a method for creating photorealistic, animatable 3D head avatars that support dynamic relighting under diverse illumination conditions. It extends the Gaussian Avatar Reconstruction family by decoupling geometry and appearance from the capture lighting, allowing the avatar to be rendered under arbitrary light sources. Unlike standard Gaussian Avatars which bake lighting into the spherical harmonics or color attributes, this method explicitly models the BRDF, enabling physically plausible relighting.

### [NoPoSplat](1484-noposplat.md)
NoPoSplat is a 3D Gaussian Splatting method that eliminates the requirement for known camera poses during reconstruction. It addresses the critical limitation of pose-dependent methods by anchoring the canonical coordinate system to a single input view and introducing intrinsics token embeddings to resolve scale ambiguity. This approach achieves superior novel view synthesis quality, particularly in scenarios with minimal view overlap.

### [SMPL-based rigid/non-rigid deformation](8385-smpl-based-rigid-non-rigid-deformation.md)
This family reconstructs animatable 3D human avatars by driving 3D Gaussian primitives with the SMPL parametric model. It solves the problem of rendering dynamic, pose-dependent cloth deformation and fine geometric details that standard static splatting cannot capture. It differs from static [Gaussian Splatting](946-gaussian-splatting.md) by predicting Gaussian parameters as functions of the model's pose, and from sibling methods like [Gaussian Avatar](1479-gaussian-avatar.md) by using explicit rigid/non-rigid transformation combinations rather than purely feed-forward networks for deformation.

### GaussianShader
- [S1, unofficial] Enriches Gaussians with material parameters such as diffusiveness, specularity, roughness, and normals.

### Codec Avatar 3D
- [S2, unofficial] A production-grade GS-based head model for volumetric telepresence by Meta.

### OmniRe
- [S2, unofficial] A holistic scene reconstruction method including humans, vehicles, and static background.

### SplatArmor
- [S3] Leverage two different MLPs to predict large motions built upon the SMPL and canonical space.
- [S3] Allow pose-dependent effects by the proposed SE(3) fields.

## Background
The concept of using Gaussian primitives for avatar reconstruction emerged as an extension of 3D Gaussian Splatting (3DGS) [S1, unofficial][S2, unofficial]. While 3DGS was originally designed for static scene reconstruction [S7, unofficial], researchers adapted it to dynamic subjects by integrating parametric models like FLAME and SMPL [S1, unofficial][S2, unofficial]. This allowed for the creation of animatable avatars that could be driven by pose and shape parameters. The field has since expanded to include robustness against transient artifacts [S1, unofficial][S2, unofficial], relightable appearance models [S1, unofficial][S2, unofficial], and pose-free training [S1, unofficial][S2, unofficial]. Production-grade implementations, such as Meta's Codec Avatars 3D, have further refined these techniques for applications like volumetric telepresence [S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
