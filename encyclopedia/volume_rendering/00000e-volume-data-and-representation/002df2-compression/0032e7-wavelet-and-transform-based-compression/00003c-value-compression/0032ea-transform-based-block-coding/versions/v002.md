# Transform-based Block Coding

Also called: Block Transform Coding, Block-based Transform Compression

## Overview
Transform-based block coding compresses volume data by partitioning it into fixed-size spatial blocks, applying a linear transform (such as DCT or DWT) to concentrate energy into a subset of coefficients, and entropy coding those coefficients. It solves the problem of high memory bandwidth requirements in volume rendering by enabling random access to compressed blocks without decompressing the entire volume. It is distinguished from its siblings by its reliance on fixed-size spatial locality and orthogonal transforms, contrasting with [Triadic frequency band decomposition](828-triadic-frequency-band-decomposition.md) which uses hierarchical subband structures, and [Vector Quantization](907-vector-quantization.md) which maps blocks to codebook entries rather than transforming them.

## Shared principles
The family operates on the principle of spatial decorrelation via linear transformation. The core mechanism involves three stages:
1.  **Block Partitioning:** The volume is divided into non-overlapping blocks of size $B \times B \times B$ (commonly $8^3$ or $4^3$).
2.  **Transform:** Each block is transformed from the spatial domain to a frequency-like domain using an orthogonal or near-orthogonal basis. Common bases include the Discrete Cosine Transform (DCT) [S1] and Discrete Wavelet Transform (DWT) [S2]. The transform concentrates the signal energy into low-frequency coefficients, allowing high-frequency coefficients (often representing noise or fine detail) to be discarded or coarsely quantized.
3.  **Quantization and Entropy Coding:** The transformed coefficients are quantized. The DC coefficient (mean value) typically receives finer quantization than AC coefficients. The quantized values are then entropy coded (e.g., using range coding or arithmetic coding) [S1].

For random access, the bitstream is structured such that each block's data is self-contained or can be located via an index. This allows a renderer to fetch and decompress only the blocks intersected by the viewing ray, avoiding the overhead of global decompression [S1].

## Subtopics

### [TSDF Block Transform Coding](2548-tsdf-block-transform-coding.md)
[TSDF Block Transform Coding](2548-tsdf-block-transform-coding.md) is a lossy compression method for Truncated Signed Distance Function (TSDF) volumes, designed to enable real-time streaming of dynamic implicit surfaces. It is distinguished from its siblings by its use of a fixed, data-independent 3D DCT (avoiding the overhead of learned transforms like KLT) and its specific strategy of grouping coefficients by DCT index across all active blocks to optimize probability models for range coding.

### [JPEG2000](2883-jpeg2000.md)
[JPEG2000](2883-jpeg2000.md) is a wavelet-based compression standard used as a 2D baseline method for volume data, where each 2D layer of a layered image is coded separately using JPEG2000 Part-1. It serves as a comparative benchmark against 3D volumetric schemes to evaluate the impact of inter-layer correlation. Unlike its sibling [TSDF Block Transform Coding](2548-tsdf-block-transform-coding.md), which focuses on specific band structures, JPEG2000 provides a general-purpose framework with features such as lossy-to-lossless performance, scalability, and region of interest (ROI) support.

### ZFP Block-Based Compression
**Definition:** A floating-point compression algorithm using fixed-rate compression, encoding blocks of $4 \times 4 \times 4$ floating-point values. It is a de facto standard in the HPC community [S1].
**Steps:**
1.  Convert block values to a common fixed-point format by factoring out the largest exponent of floating-point values within each block [S1].
2.  Convert resulting normalized values to a two's complement fixed-point format [S1].
3.  Spatially decorrelate blocks by converting to a different basis based on a tensor product transformation (e.g., DWT, DCT) [S1].
4.  Encode the resulting transform coefficients per bit plane [S1].
**Pitfall:** Decompression is only efficient when using caches. For Monte Carlo rendering with incoherent memory access, it requires ray wavefronts, barrier synchronization, and sorting, resulting in overly complex control flow [S1].

### Block Transformation Coding
**Definition:** A technique based on block transformation coding to address difficulties of rendering large datasets on a GPU [S2].

### Learned Transform Coding
**Definition:** A compression approach used when direct rate-distortion optimization is intractable. It transforms input data $x$ into a latent representation $z$ via an encoder $E$, recovers a lossy version $\hat{x}$ from a quantized latent $\hat{z}$ via a decoder $D$, and optimizes both transformations jointly with a learned probability model for the latent [S3].
**Steps:**
1.  Encode: Compute latent $z = E(x; \theta_e)$ [S3].
2.  Quantize: Discretize $z$ to $\hat{z} = Q(z)$ [S3].
3.  Model: Compute bit rate $R(\hat{z}; \phi) = -\log p_{\hat{z}}(\hat{z}; \phi)$ [S3].
4.  Decode: Recover $\hat{x} = D(\hat{z}; \theta_d)$ [S3].
5.  Entropy code $\hat{z}$ using the learned prior $p_{\hat{z}}$ to produce variable-length bit string [S3].
**Parameters:**
*   $\lambda$: rate-distortion trade-off parameter. Higher values yield better bit rates at the expense of increased distortion [S3].
*   $p_{\hat{z}}$: learned probability model (prior) for the quantized latent $\hat{z}$, with parameters $\phi$ [S3].

## Background
Transform-based block coding has been a staple in image and video compression (e.g., JPEG, MPEG) for decades. In volume rendering, it was adapted to handle the larger data sizes of 3D datasets. Early work by Fout and Ma proposed real-time volume compression and decompression based on block transformation coding to address GPU rendering difficulties [S2]. More recently, methods like TSDF Block Transform Coding have been developed specifically for dynamic implicit surfaces to enable streaming [S1]. The family also includes general-purpose standards like JPEG2000 applied to layered volumes [S1, S2] and HPC-oriented formats like ZFP [S1]. Learned transform coding represents a modern extension using neural networks for the transform and probability modeling [S3].

## Sources
[S1] GPU Volume Rendering with Hierarchical Compression Using VDB, https://arxiv.org/pdf/2504.04564, fetched 2026-10-26
[S2] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S3] Deep Implicit Volume Compression, https://doi.org/10.1109/cvpr42600.2020.00137, fetched 2026-10-26
