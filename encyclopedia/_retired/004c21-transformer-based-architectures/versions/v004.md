# Transformer-based Architectures

Also called: Transformer NeRF, ViT-NeRF, Generalizable NeRF with Transformers

## Overview
This family replaces or augments the standard Multi-Layer Perceptron (MLP) in radiance field optimization with Transformer architectures to model long-range spatial dependencies and aggregate multi-view information. Unlike sibling MLP-based methods that rely on local positional encoding and feed-forward processing, Transformer-based methods utilize self-attention mechanisms to capture global context, which is particularly effective for generalizable rendering from sparse views or for super-resolution tasks where large-scale structures must be recovered [S2][S3, unofficial].

## Shared principles
The core mechanism remains the parametric function $F_\Theta$ that maps a 3D position $\mathbf{x} \in \mathbb{R}^3$ and a viewing direction $\mathbf{d} \in \mathbb{R}^3$ to a volume density $\sigma \in \mathbb{R}$ and an RGB color $\mathbf{c} \in \mathbb{R}^3$. This function is implemented as one or more neural networks, typically utilizing positional encoding $\gamma(\mathbf{x})$ to map raw coordinates to a higher-dimensional space using sinusoidal functions at increasing frequencies [Parent].

**Rendering Integration:**
During training, rays are sampled at points $\mathbf{x}_i$ along the ray. The networks predict $\sigma_i$ and $\mathbf{c}_i$. The pixel color $C(\mathbf{r})$ is computed via volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
and $\delta_i$ is the distance between samples [Parent].

**Input Representation:**
Raw coordinates are rarely used directly. They are mapped to a higher-dimensional space using positional encoding $\gamma(\mathbf{x})$. For a coordinate $\mathbf{x}$, the encoding includes the original value and sinusoidal functions at increasing frequencies:
$$
\gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
$$
where $L$ is the number of frequency bands [Parent].

## Subtopics

### Transformer Proposer Architecture
A proposer network designed to reduce computational cost by using a single transformer block for both encoder and decoder with a single attention head [S1, unofficial]. Features are projected down to 16-D to keep the cost reasonable, as the lack of pooling operations makes the transformer somewhat more expensive than fully connected layers [S1, unofficial]. It is used in conjunction with coarse and fine networks, where the proposer handles a subset of samples (e.g., $N_c=64$) to guide sampling, adding negligible overhead to the total vector processing load [S1, unofficial].

### Generalizable Radiance Fields via Transformer Attention
A generalizable approach that uses a transformer-based attention module to aggregate source view information, enabling the learning of geometric priors from source images [S2]. This allows the model to generalize to unseen objects or scenes, often leveraging deep multiview stereo (MVS) strategies to compute cost volumes or perform feature matching using attention mechanisms [S2]. These methods are designed to work effectively with very sparse inputs (as few as two images) by leveraging geometric cues from similarities across different views [S2].

### Masked Auto Encoder (MAE) for Generalizable NeRF
A method that combines Masked Auto Encoder (MAE) with a 3D transformer-based architecture to learn a generalized representation capable of reconstructing unseen scenes [S2]. This approach facilitates transfer learning and self-supervised NeRF training, although it requires significant time on high-performance GPUs (usually several days) to learn the correspondence between 3D geometry and 2D views in advance [S2].

### Transformers and Vision Transformers (ViT)
The Transformer computes pairwise attention over input tokens, capturing long-range dependencies that convolutions miss [S3, unofficial]. The Vision Transformer (ViT) adapts this for images by treating fixed-size patches as tokens [S3, unofficial]. In volume rendering, these architectures appear in volume super-resolution (SR) and denoising, where their ability to attend across the full spatial extent of a subvolume helps recover large-scale structures that local convolutions miss [S3, unofficial].

## Background
Transformer architectures originated in natural language processing but have been adapted for 3D vision tasks. In the context of volume rendering, they were introduced to address limitations of MLPs in capturing global context and to enable generalization from sparse views. The application of ViTs to volume data specifically targets super-resolution and denoising, where global attention is beneficial for recovering large-scale structures [S3, unofficial]. Generalizable NeRF methods using transformers emerged to allow rendering from unseen scenes without per-scene optimization, often requiring pre-training on large datasets [S2].

## Sources
[S1] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
[S2] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S3] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
