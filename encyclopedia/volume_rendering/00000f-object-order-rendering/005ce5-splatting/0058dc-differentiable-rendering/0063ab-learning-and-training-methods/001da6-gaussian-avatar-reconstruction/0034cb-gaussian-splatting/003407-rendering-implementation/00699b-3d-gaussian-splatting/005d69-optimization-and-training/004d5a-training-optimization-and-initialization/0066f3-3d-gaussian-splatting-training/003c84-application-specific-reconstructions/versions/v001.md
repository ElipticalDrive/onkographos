# Application-Specific Reconstructions

Also called: Domain-Specific 3DGS, Specialized Gaussian Optimization

## Overview
Application-Specific Reconstructions extend the base 3D Gaussian Splatting (3DGS) optimization framework to handle specialized domains such as human avatars, dynamic bodies, robotics, and plants, which require structural priors or modified training pipelines beyond standard static scene reconstruction. This family is distinguished from general optimization techniques by its reliance on domain-specific constraints (e.g., mesh embedding, physical priors, or semantic features) to resolve ambiguities inherent in monocular or dynamic inputs, whereas [Base 3DGS Optimization](7549-base-3dgs-optimization.md) focuses on generic photometric loss and densification.

## Shared principles
The shared principles of this family are defined by the integration of external domain knowledge into the Gaussian parameter space (position, covariance, opacity, spherical harmonics, or feature fields). Unlike [Base 3DGS Optimization](7549-base-3dgs-optimization.md), which treats Gaussians as independent entities optimized solely for photometric consistency, application-specific methods impose structural or physical constraints to regularize the solution space.

Common mechanisms include:
1.  **Structural Embedding**: Mapping Gaussians to a canonical representation (e.g., a mesh surface or a parametric body model) to ensure topological consistency and enable motion control [S1].
2.  **Physical/Geometric Priors**: Using physical laws (e.g., cloth dynamics) or geometric constraints (e.g., surface normals) to regularize Gaussian positions and orientations in the canonical space, preventing artifacts in dynamic or translucent scenarios [S1].
3.  **Modified Optimization Steps**: Altering the standard densification, pruning, and splitting routines to better suit the specific geometry or dynamics of the target domain, often to improve training efficiency or rendering fidelity [S1].
4.  **Feature Field Integration**: Extending the Gaussian attributes to include semantic or appearance features that can be rendered alongside color, enabling tasks like segmentation or open-vocabulary querying [S2, unofficial].

## Subtopics

### [SplattingAvatar](9556-splattingavatar.md)
A real-time human avatar rendering method that embeds 3D Gaussians onto a driving mesh (FLAME) to enable non-ambiguous motion control and high-fidelity rendering of complex geometries like eyes and hair, avoiding the noise associated with nearest-triangle sampling in canonical space.

### [SplattingAvatar](9556-splattingavatar.md)
A variant that re-designs the prune/split/clone operations of the original 3DGS to achieve efficient optimization. It achieves minute-level training and real-time rendering (166 FPS).

### [SplattingAvatar](9556-splattingavatar.md)
A method for dynamic digital human modeling from monocular video that leverages physical-based priors to regularize the Gaussians in the canonical space. This regularization avoids artifacts in dynamic cloth that typically arise from monocular input ambiguity.

### [SplattingAvatar](9556-splattingavatar.md)
Adapts the feature rendering pipeline for robotics simulation, achieving real-time open-vocabulary segmentation by fusing 3DGS with a lightweight 2D segmentation network.

### [SplattingAvatar](9556-splattingavatar.md)
Applies 3DGS to create digital plant assets, leveraging alpha blending and feature fields to accurately reconstruct translucent leaf structures.

### Inpainting for Displaced Objects
A technique to fix black smudges and dirty textures caused by displaced objects exposing unseen areas that 3DGS cannot recover from input images [S3]. Unlike vanilla 3DGS which leaves empty spots and dirty textures blended from irrelevant kernels when objects are displaced, this method ameliorates the issue using generative AI inpainting [S3].

The procedure is as follows:
1.  Remove all Gaussian kernels of the object that may be displaced [S3].
2.  Use LaMa [61] to inpaint the rendered results [S3].
3.  Note the new colors of pixels originally in the spots and assign them to the first Gaussian kernel encountered by rays emitted from these pixels [S3].
4.  Average the recorded colors on all noted Gaussian kernels for their diffuse color [S3].
5.  Set opacity to 1 [S3].
6.  Set specular color to 0 to prevent highlights [S3].

## Background
The application of 3D Gaussian Splatting to specific domains emerged as researchers sought to address the limitations of the base method in handling dynamic, articulated, or semantically rich scenes. Early extensions focused on human avatars, where embedding Gaussians onto parametric meshes (like FLAME) provided a stable topology for motion [S1]. Subsequent work expanded to dynamic bodies using physical priors to handle cloth deformation [S1], and to robotics and plant modeling by integrating feature fields for semantic reasoning and translucent rendering [S2, S2, unofficial]. Inpainting techniques were developed to handle occlusions and displaced objects in dynamic scenes, such as fluid simulations, where standard 3DGS fails to reconstruct unseen background areas [S3].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S3] Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering, https://doi.org/10.1109/CVPR52734.2025.00057, fetched 2026-10-29
