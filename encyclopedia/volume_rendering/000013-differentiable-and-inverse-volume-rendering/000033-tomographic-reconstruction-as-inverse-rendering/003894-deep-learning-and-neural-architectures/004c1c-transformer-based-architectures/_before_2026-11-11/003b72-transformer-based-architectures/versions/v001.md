# Transformer-based Architectures
Also called: Transformer NeRF, ViT-NeRF, Generalizable Transformer NeRF

## Overview
This family encompasses radiance field models that replace or augment standard Multi-Layer Perceptrons (MLPs) with Transformer architectures to model spatial dependencies, aggregate multi-view information, or process volume tokens. Unlike sibling methods that rely on local convolutions, hash grids, or simple feature fusion, Transformer-based approaches explicitly compute pairwise attention over input tokens, capturing long-range dependencies that local operations miss [S3, unofficial]. They are primarily distinguished by their ability to generalize across scenes (via attention-based aggregation of source views) or to efficiently model large-scale structures in volume super-resolution and denoising [S2][S3, unofficial].

## Shared principles
All methods in this family compute a radiance field function $f_\theta(\mathbf{x}, \mathbf{d})$ that outputs volume density $\sigma$ and color $\mathbf{c}$ for a query point $\mathbf{x}$ and direction $\mathbf{d}$. The final pixel color $\mathbf{C}(\mathbf{r})$ for a ray $\mathbf{r}$ is computed via the standard volume rendering integral:
$$
\mathbf{C}(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{c}_i
$$
where $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$. The architectural variations lie in how $\sigma_i$ and $\mathbf{c}_i$ are generated from the input coordinates, how the network is structured (depth, width, connectivity), and what auxiliary inputs (embeddings, priors, features) are fused into the computation.

## Subtopics

### Transformer Proposer Architecture
A proposer network designed to reduce computational cost by using a single transformer block for both encoder and decoder with a single attention head [S1, unofficial]. Features are projected down to 16-D to keep the cost reasonable, as the lack of pooling operations makes the transformer somewhat more expensive than alternatives [S1, unofficial]. It is used in conjunction with coarse and fine networks where the proposer handles a subset of vectors (e.g., $N_c=64$) to guide sampling, adding negligible overhead to the total computation which already involves large fully connected layers [S1, unofficial].

### Generalizable Radiance Fields via Transformer Attention
A generalizable approach that uses a transformer-based attention module to aggregate source view information, allowing the model to learn geometric priors from a set of source images [S2]. This enables the representation to be fine-tuned or adapted to unseen objects or scenes, leveraging geometric cues from similarities across different views to improve geometry estimation even with sparse inputs (as few as two images) [S2].

### Masked Auto Encoder (MAE) for Generalizable NeRF
A method that combines Masked Auto Encoder (MAE) with a 3D transformer-based architecture to learn a generalized representation [S2]. This approach is capable of reconstructing unseen scenes and supports transfer learning, bringing generalization capacities to standard representations [S2]. However, it requires significant training time on high-performance GPUs (usually several days) to learn the correspondence between 3D geometry and 2D views in advance [S2].

### Transformers and Vision Transformers (ViT)
The Transformer computes pairwise attention over input tokens, capturing long-range dependencies that convolutions miss [S3, unofficial]. The Vision Transformer (ViT) adapts this for images by treating fixed-size patches as tokens [S3, unofficial]. In the context of volume rendering, these architectures appear in volume super-resolution (SR) and denoising, where their ability to attend across the full spatial extent of a subvolume helps recover large-scale structures that local convolutions miss [S3, unofficial].

## Background
The Transformer architecture, originally introduced for sequence modeling, computes pairwise attention over input tokens [S3, unofficial]. Its adaptation to visual data via Vision Transformers (ViT) has influenced 3D representations. In volume rendering, these architectures were adopted to address limitations in modeling long-range dependencies and to enable generalization across scenes without per-scene optimization [S2][S3, unofficial]. The "Transformer Proposer" variant was introduced to optimize sampling efficiency within the NeRF framework [S1, unofficial]. Generalizable transformer methods emerged to handle sparse input views and cross-dataset generalization, often leveraging multi-view stereo cues [S2].

## Sources
[S1] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
[S2] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S3] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
