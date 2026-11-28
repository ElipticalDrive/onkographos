# Specific Model Architectures and Extensions

Also called: NSVF Model Architectures

## Overview
This family defines specific architectural configurations for the density and color networks within a Neural Scene Volume Field (NSVF), primarily through the NSVF framework [S1, unofficial]. It addresses the problem of configuring input embeddings and network topology to balance reconstruction quality and training efficiency. It sets itself apart from the parent [Scene Representation and Architecture](11368-scene-representation-and-architecture.md) by providing concrete, parameterized variants (e.g., `nsvf_emb0`, `nsvf_xyzn_old`) that dictate exactly which features (position, normal, sigma, ray direction) are concatenated into the texture and density networks [S1, unofficial].

## Shared principles
The family operates on the canonical NSVF structure where two coupled MLPs map 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ to density $\sigma$ and color $\mathbf{c}$ [S1, unofficial]. The specific variations are defined by the `inputs_to_density` and `inputs_to_texture` configuration strings, which specify the source, index, and dimension of features concatenated into the network inputs [S1, unofficial].

Common feature sources include:
*   `feat`: Intermediate feature vector from the density network's hidden layers.
*   `pos`: Raw or encoded 3D position.
*   `normal`: Surface normal vector.
*   `sigma`: Volume density prediction.
*   `ray`: Viewing direction.
*   `emb`: Voxel or positional embeddings.

The syntax `source:layer:dim` indicates the feature source, the layer index from which to extract it, and the dimensionality [S1, unofficial].

## Subtopics

### [Mip-NeRF 360](3306-mip-nerf-360.md)
Mip-NeRF 360 extends the quality of neural radiance fields to unbounded scenes where the camera may rotate 360 degrees and content exists at arbitrary distances. It sets itself apart from its parent [NeRF Training Loop](970-nerf-training-loop.md) and sibling [Mip-NeRF](8585-specific-model-architectures.md) by combining a non-linear scene parameterization (contract mapping) to handle infinite space, anti-aliasing via cone-casting to resolve multi-scale details, and online distillation using a proposal network to maintain training efficiency despite increased model capacity.

### [NeRF-FF](8669-nerf-ff.md)
NeRF-FF is a plug-in strategy that mitigates the degradation of Neural Radiance Field (NeRF) reconstruction quality caused by out-of-focus (defocus blurred) regions in input images. It estimates binary image masks based on "Focus Frustums"—the visible volume in scene space that is in-focus—and uses these masks to omit out-of-focus pixels during the training of a subsequent NeRF model. Unlike methods that employ dynamic ray generation or explicit blur kernels, NeRF-FF relies on static ray assumptions, making it compatible with high-performance, runtime-optimized NeRF variants such as Instant-NGP, nerfacto, and splatfacto.

### [Drone-NeRF](8972-drone-nerf.md)
Drone-NeRF is a specialized radiance-field optimization method for reconstructing large-scale 3D scenes from aerial drone imagery, addressing challenges such as unbounded space representation, varying camera exposure, and shadow artifacts inherent in overhead views. It distinguishes itself from standard NeRF variants by employing a block-parallel training strategy with spatial contraction, appearance embeddings for exposure balancing, and a post-training merging process that explicitly mitigates shadow occlusions from vertical bounding box faces.

### nsvf_xyzn_old
A baseline NSVF configuration where the texture network input concatenates features from layer 0 of the density network (256 dims), surface normal (3 dims), density sigma (1 dim), and ray direction (4 dims) [S1, unofficial]. The density network uses 6 hidden layers with skip connections every 3 layers, and Layer Normalization is disabled [S1, unofficial].

### nsvf_xyzn_nope
A variant that removes the positional encoding from the texture network input. The texture network input concatenates features from layer 0 of the density network (256 dims), raw position (3 dims), surface normal (3 dims), density sigma (1 dim), and ray direction (4 dims) [S1, unofficial]. This contrasts with `nsvf_xyzn_old` which uses encoded position implicitly via the feature vector or explicit encoding not listed in the texture input string.

### nsvf_xyzn_noz
A configuration that excludes the density sigma from the texture network input. The texture network input concatenates encoded position (10 dims), surface normal (4 dims), and ray direction (4 dims) [S1, unofficial]. This reduces the dimensionality of the color prediction network's input compared to variants including sigma.

### nsvf_embn
A configuration utilizing voxel embeddings for the density network. The density network input concatenates embeddings from layer 6 (32 dims). The texture network input concatenates features from layer 0 of the density network (256 dims), surface normal (4 dims), and ray direction (4 dims) [S1, unofficial]. This leverages explicit spatial hashing or voxel features to accelerate training.

### nsvf_emb0
A configuration using voxel embeddings at the input layer. The voxel embedding dimension is set to 384. The density network input concatenates embeddings from layer 0 (384 dims). The texture network input concatenates features from layer 0 of the density network (256 dims) and ray direction (4 dims) [S1, unofficial]. This variant emphasizes high-dimensional voxel features for the initial density prediction.

### ReconFusion
A minor variant that employs a Diffusion Prior to constrain unobserved views during reconstruction [S2, unofficial]. This approach uses generative priors to fill in missing geometry and texture information in areas not covered by input images, improving completeness in single-image or sparse-view settings.

## Background
The NSVF (Neural Scene Volume Field) architectures were introduced by Facebook Research as a flexible framework for configuring Neural Radiance Field components [S1, unofficial]. The specific variants (`nsvf_*`) were developed to benchmark different input feature combinations and network topologies for optimizing reconstruction quality and training speed. The family also encompasses extensions like Mip-NeRF 360 for unbounded scenes, NeRF-FF for defocus handling, and Drone-NeRF for aerial imagery, addressing specific limitations of the canonical NeRF representation.

## Sources
[S1] facebookresearch/NSVF, https://github.com/facebookresearch/NSVF, fetched 2026-10-27 (unofficial)
[S2] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
