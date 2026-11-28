# UrbanGIRAFFE

Also called: UrbanGIRAFFE

## Overview
UrbanGIRAFFE is a method for generating high-resolution, semantically controllable urban scene images by extending the GIRAFFE architecture with coarse panoptic priors. It addresses the challenge of maintaining 3D controllability and generating images exceeding 512×512 resolution under substantial camera viewpoint changes. It distinguishes itself from standard GIRAFFE by integrating semantic voxel grids and object layouts to enhance controllability for semantic layouts and large viewpoint shifts.

## How it works
UrbanGIRAFFE builds on the StyleGAN2-based architecture of GIRAFFE [S1]. The method extends the base concept by incorporating coarse panoptic priors.

**Inputs and Outputs:**
- **Inputs:** Latent variables for scene generation; coarse panoptic priors consisting of semantic voxel grids and object layouts.
- **Outputs:** High-resolution images (exceeding 512×512) of urban scenes.

**Data Structures:**
- **Semantic Voxel Grids:** A 3D grid representing semantic information of the scene.
- **Object Layouts:** Data structures defining the position and type of objects within the scene.

**Steps:**
1. Initialize the StyleGAN2-based generator architecture as used in GIRAFFE [S1].
2. Incorporate coarse panoptic priors into the generation process. These priors are defined as semantic voxel grids and object layouts [S1].
3. Use these priors to guide the generation of the image, enhancing controllability for substantial camera viewpoint changes and semantic layouts [S1].

**Parameters:**
- Resolution: Exceeds 512×512 [S1].

**Edge Cases and Pitfalls:**
- Not given in the sources.

## Subtopics

### [HiGS (Hierarchically Tiled Gaussian Splatting)](23059-higs-hierarchically-tiled-gaussian-splatting.md)
HiGS is a rendering architecture for 3D Gaussian Splatting that decouples spatial partitioning from rasterization granularity using a two-scale hierarchy.

## Operating characteristics
- **Time and Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Strengths and Limitations:** Enhances controllability for substantial camera viewpoint changes and semantic layouts compared to standard GIRAFFE [S1].
- **When to use:** When high-resolution (512×512+) generation with strong semantic and viewpoint controllability is required.
- **When not:** When 3D controllability is not the primary focus or when lower resolution is acceptable.

## Background
UrbanGIRAFFE was introduced in [261] (cited in [S1]). It follows the footsteps of FFE HD [255] and work by Chan et al. [19], which aimed to retain the 3D controllability of GIRAFFE while generating higher quality and resolution images using StyleGAN2 [95]. UrbanGIRAFFE specifically extends this by using coarse panoptic priors.

## Sources
[S1] Semantically-aware Neural Radiance Fields for Visual Scene Understanding: A Comprehensive Review, https://link.springer.com/article/10.1007/s11263-025-02663-5, fetched 2026-10-27
