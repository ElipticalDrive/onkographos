# Specific Model Architectures and Extensions

Also called: NSVF Model Architectures

## Overview
This family defines specific neural network configurations and extensions for the parametric function $F_\Theta$ in radiance-field optimization, primarily originating from the NSVF framework. It addresses the problem of configuring input representations (position, direction, features) and network topology to handle specific scene characteristics, such as appearance variations or unbounded spaces. It sets itself apart from the canonical NeRF architecture by introducing explicit appearance embeddings, varied input concatenation strategies for density and color networks, and integration with auxiliary priors like diffusion models.

## Shared principles
The core mechanism is a parametric function $F_\Theta$ that maps a 3D position $\mathbf{x} \in \mathbb{R}^3$ and a viewing direction $\mathbf{d} \in \mathbb{R}^3$ to a volume density $\sigma \in \mathbb{R}$ and an RGB color $\mathbf{c} \in \mathbb{R}^3$. This function is typically implemented as one or more neural networks.

**Input Representation:**
Raw coordinates are rarely used directly due to the high-frequency nature of natural images. They are mapped to a higher-dimensional space using positional encoding $\gamma(\mathbf{x})$. For a coordinate $\mathbf{x}$, the encoding includes the original value and sinusoidal functions at increasing frequencies:
$$
\gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
$$
where $L$ is the number of frequency bands [S1, unofficial][S3].

**Network Topology:**
The canonical implementation uses two coupled Multi-Layer Perceptrons (MLPs): a density network $F_\sigma$ and a color network $F_c$ [S1, unofficial][S3].
1.  **Density Network:** Takes the encoded position $\gamma(\mathbf{x})$ as input. It consists of $L$ hidden layers with ReLU activations and skip connections from the input to every 4th layer to preserve low-frequency information [S1, unofficial][S3]. The output is the density $\sigma$.
2.  **Color Network:** Takes the encoded position $\gamma(\mathbf{x})$, the raw viewing direction $\mathbf{d}$, and the intermediate feature vector from the density network (usually the output of the last hidden layer before the density head) as input. It predicts the color $\mathbf{c}$ using a final ReLU or sigmoid activation [S1, unofficial][S3].

**Rendering Integration:**
During training, rays are sampled at points $\mathbf{x}_i$ along the ray. The MLPs predict $\sigma_i$ and $\mathbf{c}_i$. The pixel color $C(\mathbf{r})$ is computed via volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
and $\delta_i$ is the distance between samples [S1, unofficial][S3].

## Subtopics

### [Mip-NeRF 360](3306-mip-nerf-360.md)
Mip-NeRF 360 extends the quality of neural radiance fields to unbounded scenes where the camera may rotate 360 degrees and content exists at arbitrary distances. It sets itself apart from its parent [NeRF Training Loop](970-nerf-training-loop.md) and sibling [Mip-NeRF](link-to-mip-nerf) by combining a non-linear scene parameterization (contract mapping) to handle infinite space, anti-aliasing via cone-casting to resolve multi-scale details, and online distillation using a proposal network to maintain training efficiency despite increased model capacity.

### [NeRF-FF](8669-nerf-ff.md)
NeRF-FF is a plug-in strategy that mitigates the degradation of novel view synthesis caused by out-of-focus (defocus blurred) input images by estimating binary masks that identify in-focus regions. It enables subsequent NeRF models to omit out-of-focus image regions during training, allowing the network to learn an in-focus representation of the scene. Unlike methods that use dynamic ray generation and blur kernels (e.g., PDRF, Deblur-NeRF), NeRF-FF relies on static ray assumptions, making it compatible with runtime-optimized variants like Instant-NGP, nerfacto, and splatfacto.

### [Drone-NeRF](8972-drone-nerf.md)
Drone-NeRF is a specialized volume rendering method for reconstructing large-scale 3D scenes from aerial drone imagery, addressing the challenges of unbounded space representation and varying camera exposure. It distinguishes itself from standard NeRF training by integrating view frustum sampling for anti-aliasing, a pose optimizer for joint pose refinement, and appearance embeddings to handle exposure discrepancies, while employing a spatial contraction function to map unbounded scenes into a bounded domain for efficient rendering. It builds on [Specific Model Architectures and Extensions](10278-specific-model-architectures-and-extensions.md) by configuring the MLP to accept position, direction, and appearance embeddings, and utilizes a block-parallel training strategy with a subsequent merging phase to handle large-scale overhead views.

### ReconFusion
ReconFusion is an extension that uses a Diffusion Prior to constrain unobserved views during 3D reconstruction [S2, unofficial].

## Background
The specific model architectures described herein originate from the NSVF (Neural Scene Volume Fields) framework developed by Facebook Research [S1, unofficial]. The framework registers various configurations under the 'nsvf' namespace to configure inputs to density and texture networks. The family also includes extensions like Mip-NeRF 360 for unbounded scenes [S2, unofficial] and NeRF-FF for handling defocus blur [S1, unofficial].

## Sources
[S1] facebookresearch/NSVF, https://github.com/facebookresearch/NSVF, fetched 2026-10-27 (unofficial)
[S2] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
