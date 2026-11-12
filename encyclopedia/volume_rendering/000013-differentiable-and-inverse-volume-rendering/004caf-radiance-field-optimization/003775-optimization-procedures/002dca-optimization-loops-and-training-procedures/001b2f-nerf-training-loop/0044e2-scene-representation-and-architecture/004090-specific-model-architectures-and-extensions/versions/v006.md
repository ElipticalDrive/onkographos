# Specific Model Architectures and Extensions

Also called: NSVF Model Architectures

## Overview
This family defines specific neural network topologies and input configurations for approximating radiance fields, primarily originating from the NSVF framework. It sets itself apart from the canonical [NeRF Training Loop](970-nerf-training-loop.md) by explicitly configuring the feature connections between density and texture networks, and by supporting various input encodings (such as appearance embeddings or specific coordinate subsets) to handle different scene constraints [S1, unofficial].

## Shared principles
The family shares the mechanism of approximating the continuous radiance field using a neural network or a collection of networks. The function $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$ is differentiable with respect to $\Theta$, allowing gradient-based optimization via backpropagation through the volumetric rendering integral [Parent].

**Input Representation:**
Raw coordinates $\mathbf{x}$ and $\mathbf{d}$ are rarely used directly due to the spectral bias of MLPs, which struggle to fit high-frequency details. They are mapped to a higher-dimensional space using positional encoding $\gamma(\cdot)$ [Parent, S1, unofficial, S3]. For a coordinate $\mathbf{x}$, the encoding includes the original value and sinusoidal functions at increasing frequencies:
$$
\gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
$$
where $L$ is the number of frequency bands [Parent, S1, unofficial, S3].

**Network Topology:**
The canonical implementation uses two coupled Multi-Layer Perceptrons (MLPs): a density network $F_\sigma$ and a color network $F_c$ [Parent, S1, unofficial, S3].
1.  **Density Network:** Takes the encoded position $\gamma(\mathbf{x})$ as input. It consists of $L$ hidden layers with ReLU activations and skip connections from the input to every 4th layer to preserve low-frequency information [Parent, S1, unofficial, S3]. The output is the density $\sigma$.
2.  **Color Network:** Takes the encoded position $\gamma(\mathbf{x})$, the raw viewing direction $\mathbf{d}$, and the intermediate feature vector from the density network (usually the output of the last hidden layer before the density head) as input. It predicts the color $\mathbf{c}$ using a final ReLU or sigmoid activation [Parent, S1, unofficial, S3].

**Rendering Integration:**
During training, rays are sampled at points $\mathbf{x}_i$ along the ray. The MLPs predict $\sigma_i$ and $\mathbf{c}_i$. The pixel color $C(\mathbf{r})$ is computed via volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
and $\delta_i$ is the distance between samples [Parent, S1, unofficial, S3].

## Subtopics

### [Mip-NeRF 360](3306-mip-nerf-360.md)
Mip-NeRF 360 extends the quality of neural radiance fields to unbounded scenes where the camera may rotate 360 degrees and content exists at arbitrary distances. It sets itself apart from its parent [NeRF Training Loop](970-nerf-training-loop.md) and sibling [Mip-NeRF](link-to-mip-nerf) by combining a non-linear scene parameterization (contract mapping) to handle infinite space, anti-aliasing via cone-casting to resolve multi-scale details, and online distillation using a proposal network to maintain training efficiency despite increased model capacity.

### [NeRF-FF](8669-nerf-ff.md)
NeRF-FF is a plug-in strategy that mitigates the degradation of Neural Radiance Field (NeRF) reconstruction quality caused by out-of-focus (defocus blurred) regions in input images. It solves the problem of training NeRFs on partially blurred data by estimating binary image masks that identify in-focus regions, allowing the subsequent NeRF model to omit out-of-focus pixels during training. Unlike methods that use dynamic ray generation or blur kernels (e.g., PDRF, Deblur-NeRF), NeRF-FF relies on static ray assumptions, making it compatible with runtime-optimized NeRF variants like Instant-NGP, nerfacto, and splatfacto.

### [Drone-NeRF](8972-drone-nerf.md)
Drone-NeRF is a specialized radiance-field optimization method for reconstructing large-scale 3D scenes from aerial drone imagery, addressing challenges such as unbounded space representation, varying camera exposure, and shadow artifacts caused by vertical scene boundaries. It distinguishes itself from standard NeRF training by integrating appearance embeddings to handle exposure discrepancies, a pose optimizer for joint refinement, and a block-parallel training strategy with a specific shadow-mitigation merging process for overhead views.

### nsvf_xyzn_old
A specific NSVF configuration where the texture network inputs are defined as `feat:0:256, normal:0:3, sigma:0:1, ray:4`. It uses 6 feature layers, skips connections every 3 layers, disables layer normalization in the MLP, and does not use positional encoding for inputs [S1, unofficial].

### nsvf_xyzn_nope
A variant where positional encoding is omitted from the texture network inputs. The texture network inputs are defined as `feat:0:256, pos:0:3, normal:0:3, sigma:0:1, ray:4` [S1, unofficial].

### nsvf_xyzn_noz
A variant where the Z-coordinate is excluded from the positional encoding inputs to the texture network. The texture network inputs are defined as `pos:10, normal:4, ray:4` [S1, unofficial].

### nsvf_embn
A configuration that uses appearance embeddings for the density network. The density network inputs are `emb:6:32`, and the texture network inputs are `feat:0:256, normal:4, ray:4` [S1, unofficial].

### nsvf_emb0
A configuration with a voxel embedding dimension of 384. The density network inputs are `emb:0:384`, and the texture network inputs are `feat:0:256, ray:4` [S1, unofficial].

### ReconFusion
A minor variant that uses a Diffusion Prior to constrain unobserved views during reconstruction [S2, unofficial].

## Background
The NSVF (Neural Scene Volume Fields) model architectures were introduced by Facebook Research as a flexible framework for configuring inputs to density and texture networks [S1, unofficial]. The specific variants (nsvf_emb0, nsvf_embn, etc.) allow for experimentation with different input representations such as appearance embeddings and coordinate subsets. Mip-NeRF 360 was developed to address the limitations of standard NeRF in unbounded, 360-degree scenes [S2, unofficial]. NeRF-FF was proposed to handle defocus blur in input imagery [S1, unofficial]. Drone-NeRF was designed specifically for aerial drone data [S1, unofficial]. ReconFusion utilizes diffusion priors for 3D reconstruction from limited views [S2, unofficial].

## Sources
[S1] facebookresearch/NSVF, https://github.com/facebookresearch/NSVF, fetched 2026-10-27 (unofficial)
[S2] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
