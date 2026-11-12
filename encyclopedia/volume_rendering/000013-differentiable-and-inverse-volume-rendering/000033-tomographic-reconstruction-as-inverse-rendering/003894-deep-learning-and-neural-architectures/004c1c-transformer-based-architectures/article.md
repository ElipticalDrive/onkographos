# Transformer-based Architectures


## Overview
Transformer-based architectures apply self-attention mechanisms to volume rendering and tomographic reconstruction, enabling the modeling of long-range dependencies that local convolutions miss [S1][S5]. They address the ill-posed nature of inverse problems (e.g., sparse-view CT, limited-angle SPECT) and the computational cost of per-scene optimization in NeRFs by leveraging global context aggregation [S1][S2, unofficial]. This family distinguishes itself from sibling methods like [Deep Image Prior Reconstruction](1221-deep-image-prior-reconstruction.md) and [R2-Gaussian](6090-r2-gaussian.md) by replacing iterative optimization or explicit geometric priors with learned attention-based feature fusion and reconstruction [S1][S2, unofficial].

## Shared principles
The family shares the foundational mechanism of using self-attention to aggregate information across spatial or ray-based tokens. Unlike convolutional operations that rely on local receptive fields, transformers compute pairwise interactions between all tokens, allowing the model to capture global structural context [S1][S5]. In the context of volume rendering, this often involves:
1.  **Tokenization:** Discretizing input data (images, projections, or rays) into patches or points [S1][S5].
2.  **Attention Aggregation:** Using Query, Key, and Value projections to compute attention weights that fuse information from multiple views or along ray depths [S1][S5].
3.  **Global Context Modeling:** Overcoming the limitations of CNNs where local receptive fields are insufficient for understanding the whole image or scene geometry [S1][S2, unofficial].

## Subtopics

### [DuDoTrans](1409-dudotrans.md)
A dual-domain transformer using Swin Transformer blocks with window-based multi-head self-attention (W-MSA) and shifted windows for sparse-view CT reconstruction. It reduces computational complexity by restricting attention to local windows while capturing global context through hierarchical shifting. Choose this for sparse-view CT where computational efficiency and local-global feature integration are critical.

### [Generalizable Radiance Fields via Transformer Attention](8996-generalizable-radiance-fields-via-transformer-attention.md)
Employs a transformer-based attention module to aggregate source view features for novel view synthesis, learning generalizable geometric priors without per-scene optimization. Choose this for generalizable NeRF applications where rapid adaptation to unseen scenes is required.

### [Transformers and Vision Transformers (ViT)](10441-transformers-and-vision-transformers-vit.md)
Adapts the standard Vision Transformer to volume rendering tasks like super-resolution and denoising by treating image patches as tokens. Choose this for tasks requiring global spatial reasoning over subvolumes, such as recovering large-scale structures.

### [Transformer-based Tomographic Reconstruction](11047-transformer-based-tomographic-reconstruction.md)
A slice-by-slice transformer (SSTrans-3D) for SPECT reconstruction that relies entirely on a transformer architecture rather than iterative optimization. Choose this for limited-angle or incomplete projection data where CNNs fail to capture global context.

### Depth-Encoded Ray Transformer (DERT)
**Definition:** A transformer architecture that infers volume density by incorporating relative depth differences as position encoding, rather than absolute positions [S1].
**Steps:**
1.  Sample $M$ points $p_m$ on ray $r$ with depths $t_m$ [S1].
2.  Calculate relative depth encoding $E_m$ for each point using $\delta_m$ [S1].
3.  Element-wise add $E_m$ to point feature $f_m$ [S1].
4.  Pass through self-attention and feed-forward network to predict volume density $\sigma_m$ [S1].
**Parameters:**
*   $K$: Length of the position encoding feature vector [S1].
*   $u$: Hyperparameter controlling the range of $\delta_m$ [S1].
**Formulas:**
The relative depth $\delta_m$ is calculated as:
$$
\delta_m = \frac{t_m - d}{u(t_{far} - t_{near})}
$$
where $d$ is the estimated depth of the ray, and $t_{near}, t_{far}$ are the near and far plane depths [S1]. The position encoding $E_m$ is:
$$
E_m = \begin{cases} \sin(\omega_k \delta_m) & \text{if } k = 2i \\ \cos(\omega_k \delta_m) & \text{if } k = 2i + 1 \end{cases}, \quad \omega_k = \frac{1}{10000^{2i/K}}
$$
[S1].

### Transformer Proposer Architecture
**Definition:** A proposer network constrained to reduce computational cost, using a single transformer block for encoder and decoder with a single attention head [S2, unofficial].
**Layout:**
*   Features projected to 16-D [S2, unofficial].
*   Single transformer block for encoder [S2, unofficial].
*   Single transformer block for decoder [S2, unofficial].
*   Single attention head [S2, unofficial].
**Context:** This architecture is used in NeRF sampling to propose points for coarse and fine networks, balancing cost and performance [S2, unofficial].

### Masked Auto Encoder (MAE) for Generalizable NeRF
**Definition:** Using Masked Auto Encoder combined with a 3D transformer-based architecture to learn a generalized representation capable of reconstructing unseen scenes [S3].
**Context:** This approach scales self-supervised NeRF training by leveraging geometric cues from similarities across different views, improving geometry estimation even with very sparse inputs (as few as two images) [S3].

### X-ray Reconstruction Transformer
**Definition:** A large feedforward transformer model that encodes sparse-view X-ray projections into compact latent tokens and fuses them across views to reconstruct a 3D CT volume density field [S4, unofficial].
**Steps:**
1.  Encode each X-ray projection $I_i$ independently using a feature extractor $F_{enc}$ (specifically DINO) to produce patch features $H_i$: $H_i = F_{enc}(I_i)$ for $i = 1, \dots, K$ [S4, unofficial].
2.  Inject camera geometry into the encoded features by applying ModLN with Plücker ray embeddings $R_i$: $\tilde{H}_i = ModLN(H_i, R_i)$ for $i = 1, \dots, K$ [S4, unofficial].
3.  Concatenate the modulated features from all $K$ views and process them through a 16-layer Fusion ViT (similar to ViT-B) using all-to-all self-attention $F_{fuse}$ to produce fused features $\{F_1, \dots, F_K\}$ [S4, unofficial].
**Parameters:**
*   Fusion ViT depth: 16 layers, architecture similar to ViT-B [S4, unofficial].
*   Encoder backbone: DINO transformer [S4, unofficial].
*   Camera embedding: Plücker ray embedding $R_i$, derived from the cross product between camera location and ray direction, independent of scale, camera position, and focal length [S4, unofficial].

### NeRFormer
**Definition:** A model using Transformers for volume rendering, processing ray-depth-ordered feature sequences from multiple source views [S5].
**Steps:**
1.  Processes ray-depth-ordered feature sequences from multiple source views [S5].
2.  Uses alternating pooling and ray-wise attention layers [S5].
3.  Enables effective joint feature aggregation and ray marching [S5].
**Context:** NeRFormer is geometry-free and does not produce the scene’s density function, nor does it rely on geometric inductive biases [S5].

## Background
Transformer architectures have been adapted for volume rendering to address limitations in CNNs, such as local receptive fields and inability to capture global context [S1][S2, unofficial]. The application of Transformers to NeRFs began with methods like NeRFormer, which used ray-wise attention for view synthesis [S5]. Subsequent works introduced generalizable approaches using attention to aggregate multi-view features, reducing the need for per-scene optimization [S1][S3]. In tomographic reconstruction, Transformers like DuDoTrans and SSTrans-3D have been applied to sparse-view and limited-angle problems, leveraging self-attention to mitigate artifacts caused by incomplete data [S1][S2, unofficial]. The use of relative depth encoding (DERT) and masked autoencoders (MAE) has further enhanced the ability of Transformers to handle varying scene scales and generalize to unseen data [S1][S3].

## Sources
[S1] NeRF-SDP: Efficient Generalizable Neural Radiance Field with Scene Depth Perception, https://doi.org/10.1145/3595916.3626380, fetched 2026-10-30
[S2] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
[S3] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S4] X-GRM: Large Gaussian Reconstruction Model for Sparse-view X-rays to Computed Tomography, https://doi.org/10.48550/arXiv.2505.15235, fetched 2026-10-27 (unofficial)
[S5] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
