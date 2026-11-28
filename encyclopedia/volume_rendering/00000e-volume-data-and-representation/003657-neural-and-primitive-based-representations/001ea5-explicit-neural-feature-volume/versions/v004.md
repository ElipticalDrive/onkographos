# Explicit Neural Feature Volume

Also called: Explicit Neural Feature Tensor, Large Tensor Storage

## Overview
Explicit Neural Feature Volume stores neural feature data as a large, directly addressable tensor, distinguishing itself from hybrid explicit-implicit models that compress features into secondary neural networks [S1]. It solves the problem of intermediate memory overhead and reconstruction artifacts in limited-angle tomography by avoiding the additional regularization and decoding latency inherent in implicit compression schemes [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D spatial coordinates $\mathbf{x} \in \mathbb{R}^3$ (and potentially viewing direction $\mathbf{d}$ or time $t$).
*   **Output:** A feature vector $\mathbf{f} \in \mathbb{R}^k$ retrieved from the tensor at the queried location.

**Data Structures:**
The core data structure is a dense or sparse tensor $T$ of shape $(D_x, D_y, D_z, C)$, where $D_x, D_y, D_z$ are the spatial dimensions and $C$ is the feature channel dimension. Unlike hash grids or octrees, the memory layout is contiguous or structured for direct indexing [S1].

**Steps:**
1.  **Initialization:** Allocate tensor $T$ with shape $(D_x, D_y, D_z, C)$. Initialize values (e.g., zeros or learned embeddings) [S1].
2.  **Query:** Given a coordinate $\mathbf{x}$, perform trilinear interpolation or nearest-neighbor lookup in $T$ to retrieve $\mathbf{f}$ [G].
3.  **Decoding:** Pass $\mathbf{f}$ to a decoder network (e.g., [Decoder Network](2301-decoder-network.md)) to produce density $\sigma$ and color $\mathbf{c}$ [S1].
4.  **Rendering:** Integrate $\sigma$ and $\mathbf{c}$ along rays using standard volume rendering equations [G].

**Parameters:**
*   Tensor resolution $(D_x, D_y, D_z)$: Determined by the scene extent and desired detail. Not given in the sources for a fixed value, but implied to be "large" [S1].
*   Feature dimension $C$: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Memory Consumption:** Intermediate memory consumption during training is higher than hybrid models because all feature volumes must be decoded to trace rays for one view [S1].
*   **Limited Angle Tomography:** In cases with missing viewing directions (missing wedge), explicit representations perform better than hybrid implicit-implicit models, which encourage blur in the missing direction [S1].

## Subtopics

### [Decoder Network](2301-decoder-network.md)
A neural network component that transforms a low-dimensional feature vector into a single scalar density value, acting as a regularizer to improve reconstruction quality in ill-posed or difficult conditions. It distinguishes itself from sibling methods by explicitly mapping a stored tensor feature to output via a learnable non-linear function, whereas [Density Activation](2421-density-activation.md) may use fixed or simpler activation functions, and [Explicit Neural Feature Volume](1838-explicit-neural-feature-volume.md) defines the storage mechanism rather than the decoding logic.

### [Density Activation](2421-density-activation.md)
Density Activation converts raw neural feature outputs (log-density or latent features) into physically meaningful volume density ($\sigma$) and opacity ($\alpha$) values suitable for volume rendering. It solves the problem of ensuring non-negative, differentiable density values that correctly account for the sampling interval along a ray. Unlike the parent [Explicit Neural Feature Volume](1838-explicit-neural-feature-volume.md) which stores the features, or the [Decoder Network](2301-decoder-network.md) which processes them, Density Activation is the specific mathematical transformation step that bridges latent space and rendering space. It is distinguished from [Neural Volume Construction](8453-neural-volume-construction.md) by focusing on the activation function itself rather than the grid initialization.

### [DINO ViT Feature Extraction](6425-dino-vit-feature-extraction.md)
DINO ViT Feature Extraction is a method for defining transfer functions in volume rendering by leveraging high-level semantic features extracted from volumetric data using a frozen, self-supervised pre-trained Vision Transformer (ViT). It solves the problem of locality in traditional intensity-based features by providing semantic context, allowing users to select structures of interest via slice annotations and automatically identifying similar structures based on feature similarity without requiring model training. Unlike supervised learning or generative adversarial network approaches that require costly per-volume training or large labeled datasets, this method enables interactive exploration with quick inference on consumer hardware.

### [Neural Volume Construction](8453-neural-volume-construction.md)
Neural Volume Construction is the process of generating a dense, world-aligned neural feature volume from sparse, noisy input primitives (such as Gaussian centers or point clouds) using a sparse 3D convolutional network. It solves the problem of initializing a coherent latent representation for volume-based rendering by aggregating local geometric cues into a global spatial structure. Unlike sibling methods like [TensoRF Vector-Matrix Decomposition](9759-tensorf-vector-matrix-decomposition.md) which rely on low-rank factorization, or [DINO ViT Feature Extraction](6425-dino-vit-feature-extraction.md) which uses 2D image-based encoders, this method explicitly constructs a 3D latent field that integrates both local and global information through an encoder-decoder architecture.

### [TensoRF Vector-Matrix Decomposition](9759-tensorf-vector-matrix-decomposition.md)
TensoRF Vector-Matrix Decomposition is a factorization technique for representing 4D neural feature volumes, reducing memory overhead compared to dense tensor storage. It sets itself apart from sibling methods like the [Decoder Network](2301-decoder-network.md) by explicitly decomposing the feature volume into structured low-rank components (vectors and matrices) rather than relying solely on implicit coordinate-based decoding or separate density activations.

### BackEnd Neural Network Module
A PyTorch `nn.Module` that processes point features using a PointTransformerV3, downsampling, and zero-convolution blocks, with a gating mechanism [S2, unofficial]. It uses a hash function to map 4D coordinates (batch, x, y, z) to a single 64-bit integer hash [S2, unofficial].

**Parameters:**
*   `hash_base`: Integer, default 1024. Used as a base for hashing voxel coordinates [S2, unofficial].
*   `k_neighbors`: Integer, default 16. Number of neighbors for KNN operations [S2, unofficial].

**Layout:**
*   `aligner`: Sequential module with `Linear(1024->512)->GELU->Linear(512->256)->GELU` [S2, unofficial].
*   `point_transformer`: Instance of `PointTransformerV3` [S2, unofficial].
*   `downsample`: Instance of `DownBlock` with `in_channels=out_dim, mid_channels=out_dim, out_channels=out_dim` [S2, unofficial].
*   `zero_conv`: Instance of `ZeroConvBlock` with `in_channels=out_dim, mid_channels=out_dim, out_channels=out_dim` [S2, unofficial].
*   `gate_scale`: `nn.Parameter` initialized to `ones(1)` [S2, unofficial].
*   `out_aligner`: `Linear(out_dim->in_dim)` if `out_dim != in_dim`, else `Identity`. Initialized with `normal(0, 1e-4)` weights and zero bias [S2, unofficial].

**Pitfalls:**
*   PointTransformerV3 relies on `spconv` which does not support bf16/fp16; it must remain in fp32. Other modules can run in bf16 [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** High intermediate memory consumption during training because all feature volumes need to be decoded to trace all rays for one view [S1]. Final explicit-implicit representations are more compact, but the explicit tensor itself consumes memory proportional to its resolution [S1].
*   **Time:** Training times are lower than hybrid explicit-implicit models, which add an implicit network that increases decoding latency [S1].

**Measured Performance:**
*   In limited angle tomography, explicit representations achieve higher PSNR than hybrid explicit-implicit models (like ACORN), which suffer from increased blur in the missing wedge direction [S1].

**Strengths and Limitations:**
*   **Strengths:** Avoids blur in limited-angle scenarios; faster training than hybrid models; simple data structure [S1].
*   **Limitations:** High intermediate memory usage; final storage size is larger than compressed hybrid models [S1].

**When to Use:**
*   Use when reconstruction quality in limited-angle tomography is critical and intermediate memory is available [S1].
*   Use when training speed is prioritized over final model compactness [S1].

**When Not to Use:**
*   Do not use when final model size must be minimized and training time is less critical, in favor of hybrid models [S1].
*   Do not use in standard tomography if hybrid models are preferred for their regularization properties, though explicit is still superior in limited-angle cases [S1].

## Background
**Origin:**
The method is described in "NeAT: Neural Adaptive Tomography" [S1].

**History and Development:**
The approach contrasts with hybrid explicit-implicit models like ACORN [Martel et al. 2021], which compress feature volumes into implicit neural networks [S1]. The authors found that while hybrid models offer compression, they worsen PSNR in limited angle tomography by encouraging blur in the missing wedge direction [S1].

**Where it is used:**
Neural adaptive tomography reconstruction [S1].

## Sources
[S1] NeAT: Neural Adaptive Tomography, https://doi.org/10.48550/arxiv.2202.02171, fetched 2026-10-26
[S2] Zhaoyibinn/VGVGGT, https://github.com/Zhaoyibinn/VGVGGT, fetched 2026-10-27 (unofficial)
