# Transformer-based Architectures

Also called: Transformer NeRF, ViT-NeRF, Attention-based Radiance Fields

## Overview
This family replaces or augments the standard Multi-Layer Perceptron (MLP) radiance field with Transformer architectures to capture long-range spatial dependencies and global context that local convolutions or shallow MLPs often miss [S3, unofficial]. Unlike sibling methods that primarily optimize scene-specific parameters for a single dataset, many Transformer-based approaches focus on **generalization**, enabling the reconstruction of unseen scenes from sparse input views by learning geometric priors across a dataset [S2].

## Shared principles
The core mechanism remains the differentiable approximation of the continuous radiance field $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$ [Parent]. However, the feature extraction and aggregation logic differs from the canonical MLP:

1.  **Tokenization and Positional Encoding:** Input data (images, voxels, or points) are converted into tokens. Positional encoding $\gamma(\cdot)$ is still critical to inject spatial structure, as Transformers lack the inductive bias of convolutional locality [Parent, S3].
2.  **Self-Attention Mechanism:** The Transformer computes pairwise attention over input tokens, allowing the model to capture long-range dependencies across the entire scene or subvolume [S3, unofficial]. This is particularly useful for recovering large-scale structures in volume super-resolution and denoising [S3, unofficial].
3.  **Generalization via Aggregation:** In generalizable variants, a transformer-based attention module aggregates information from multiple source views to form a unified feature volume or latent representation, enforcing consistency across views [S2].

## Subtopics

### AdaNeRF
See [AdaNeRF](6195-adanerf.md).

### SinNeRF
See [SinNeRF](6199-sinnerf.md).

### NeRF-RPN
See [NeRF-RPN](8663-nerf-rpn.md).

### AligNeRF
See [AligNeRF](8695-alignerf.md).

### MultiPlaneNeRF
See [MultiPlaneNeRF](8875-multiplanenerf.md).

### Pool proposer architecture
See [Pool proposer architecture](8953-pool-proposer-architecture.md).

### NeRDi
See [NeRDi](9112-nerdi.md).

### Neural Light Field Representations
See [Neural Light Field Representations](9273-neural-light-field-representations.md).

### RING-NeRF
See [RING-NeRF](9323-ring-nerf.md).

### Sparse and Efficient Representations
See [Sparse and Efficient Representations](9326-sparse-and-efficient-representations.md).

### Geometric Priors and Depth Integration
See [Geometric Priors and Depth Integration](9395-geometric-priors-and-depth-integration.md).

### Generative and Style-based Models
See [Generative and Style-based Models](9572-generative-and-style-based-models.md).

### IRON
See [IRON](10251-iron.md).

### pi-GAN
See [pi-GAN](10298-pi-gan.md).

### Instance-NeRF
See [Instance-NeRF](10611-instance-nerf.md).

### MIMO-NeRF
See [MIMO-NeRF](10928-mimo-nerf.md).

### Appearance and Lighting Decomposition
See [Appearance and Lighting Decomposition](11061-appearance-and-lighting-decomposition.md).

### Quadri-Plane Radiance Fields
See [Quadri-Plane Radiance Fields](11128-quadri-plane-radiance-fields.md).

### GNeuVox
See [GNeuVox](11357-gneuvox.md).

### HexPlane
See [HexPlane](11358-hexplane.md).

### STensoRRF
See [STensoRRF](11359-stensorrf.md).

### GIRAFFE
See [GIRAFFE](11360-giraffe.md).

### Ced-NeRF
See [Ced-NeRF](11364-ced-nerf.md).

### Dynamic and Deformable Fields
See [Dynamic and Deformable Fields](11366-dynamic-and-deformable-fields.md).

### Transformer Proposer Architecture
A proposer network designed to reduce computational cost during ray sampling. It uses a single transformer block for both encoder and decoder with a single attention head [S1, unofficial]. Features are projected down to 16-D to keep the overhead negligible compared to the main network's fully connected layers [S1, unofficial]. While slightly more expensive than pooling-based proposers due to the lack of pooling operations, this constrained design makes the cost reasonable [S1, unofficial].

### Generalizable Radiance Fields via Transformer Attention
A generalizable approach that uses a transformer-based attention module to aggregate source view information [S2]. This allows the model to learn generalizable geometric priors from source images, enabling reconstruction of unseen objects or scenes [S2]. It leverages geometric cues from similarities across different views, improving geometry estimation even with very sparse inputs (as few as two images) [S2].

### Masked Auto Encoder (MAE) for Generalizable NeRF
Uses a Masked Auto Encoder combined with a 3D transformer-based architecture to learn a generalized representation [S2]. This approach is capable of reconstructing unseen scenes and supports transfer learning [S2]. However, it requires significant training time on high-performance GPUs (usually several days) to learn the correspondence between 3D geometry and 2D views in advance [S2].

### Transformers and Vision Transformers (ViT)
The Transformer computes pairwise attention over input tokens, capturing long-range dependencies that convolutions miss [S3, unofficial]. The Vision Transformer (ViT) adapts this for images by treating fixed-size patches as tokens [S3, unofficial]. In volume rendering, Transformers appear in volume super-resolution and denoising, where their ability to attend across the full spatial extent of a subvolume helps recover large-scale structures that local convolutions miss [S3, unofficial].

## Background
The application of Transformers to volume rendering and NeRFs emerged as researchers sought to overcome the limitations of MLPs (spectral bias) and CNNs (local receptive fields) in modeling global scene structure. Early work introduced Transformer blocks as "proposers" to optimize sampling strategies [S1, unofficial]. Subsequently, the focus shifted to generalizable NeRFs, where attention mechanisms were used to aggregate multi-view features for zero-shot or few-shot reconstruction [S2]. The integration of Vision Transformers (ViT) has also been explored for volume super-resolution and denoising, leveraging their strength in capturing long-range dependencies [S3, unofficial].

## Sources
[S1] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
[S2] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S3] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
