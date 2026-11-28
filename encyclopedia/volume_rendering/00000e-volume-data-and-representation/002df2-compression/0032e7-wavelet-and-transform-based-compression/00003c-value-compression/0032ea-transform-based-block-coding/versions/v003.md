# Transform-based Block Coding
## Overview
Transform-based block coding compresses volumetric data by partitioning it into fixed-size spatial blocks, applying a linear transform (such as DCT or DWT) to concentrate energy into a subset of coefficients, and entropy coding the retained values [S1]. It differs from its siblings by operating on localized spatial neighborhoods rather than global frequency bands or learned latent spaces, making it suitable for random access patterns in GPU rendering [S1].
## Shared principles
The family shares a common pipeline:
1.  **Block Partitioning:** The volume is divided into non-overlapping blocks of size $B \times B \times B$ (commonly $8^3$ or $4^3$).
2.  **Transform:** Each block undergoes a linear transform $T$ to produce coefficients $C$. For orthogonal transforms like DCT, the basis functions are fixed. For DWT, lifting steps are often used for integer-to-integer mapping [S2].
3.  **Coefficient Selection:** A subset of coefficients is retained based on frequency significance (e.g., low-frequency indices) or a fixed rate budget. The remaining coefficients are discarded or set to zero.
4.  **Quantization:** Retained coefficients are quantized. DC terms often use a fixed step $\Delta_{DC}$, while AC terms use a tunable step $\Delta_{AC}$ [S1].
5.  **Entropy Coding:** Quantized integers are encoded using adaptive models (e.g., range coding or arithmetic coding) [S1].
6.  **Reconstruction:** Inverse transform is applied to the quantized coefficients to recover the block [S1].
## Subtopics

### [TSDF Block Transform Coding](2548-tsdf-block-transform-coding.md)
[TSDF Block Transform Coding](2548-tsdf-block-transform-coding.md) is a lossy compression method for Truncated Signed Distance Function (TSDF) volumes, designed to enable real-time streaming of dynamic implicit surfaces. It solves the problem of efficiently encoding sparse, locally smooth TSDF data by decomposing the volume into fixed-size voxel blocks, applying a 3D Discrete Cosine Transform (DCT) to concentrate energy in low frequencies, and entropy coding the retained coefficients. It is distinguished from its siblings by its use of a fixed, data-independent 3D DCT (avoiding the overhead of learned transforms like KLT) and its specific strategy of grouping coefficients by DCT index across all active blocks to optimize probability models for range coding.

### [JPEG2000](2883-jpeg2000.md)
[JPEG2000](2883-jpeg2000.md) is a wavelet-based compression standard used as a 2D baseline method for volume data, where each 2D layer of a layered image is coded separately using JPEG2000 Part-1. It serves as a comparative benchmark against 3D volumetric schemes to evaluate the impact of inter-layer correlation. Unlike its sibling [TSDF Block Transform Coding](2548-tsdf-block-transform-coding.md), which focuses on specific band structures, JPEG2000 provides a general-purpose framework with features such as lossy-to-lossless performance, scalability, and region of interest (ROI) support.

### [Learned Transform Coding](7046-learned-transform-coding.md)
[Learned Transform Coding](7046-learned-transform-coding.md) addresses the intractability of direct rate-distortion optimization for high-dimensional data by replacing fixed linear transforms with trainable neural networks. Unlike its siblings [Transform-based Block Coding](6882-transform-based-block-coding.md), [TSDF Block Transform Coding](2548-tsdf-block-transform-coding.md), and [JPEG2000](2883-jpeg2000.md) which rely on fixed bases like DCT or DWT, this method learns an encoder-decoder pair and a probability model jointly to minimize distortion and bit rate.

#### ZFP Block-Based Compression
**Definition:** A floating-point compression algorithm using fixed-rate compression, encoding blocks of $4 \times 4 \times 4$ floating-point values. It is a de facto standard in the HPC community [S1].
**Steps:**
1.  Convert block values to a common fixed-point format by factoring out the largest exponent of floating-point values within each block [S1].
2.  Convert resulting normalized values to a two's complement fixed-point format [S1].
3.  Spatially decorrelate blocks by converting to a different basis based on a tensor product transformation (e.g., DWT, DCT) [S1].
4.  Encode the resulting transform coefficients per bit plane [S1].
**Pitfall:** Decompression is only efficient when using caches. For Monte Carlo rendering with incoherent memory access, it requires ray wavefronts, barrier synchronization, and sorting, resulting in overly complex control flow [S1].

#### Block Transformation Coding
**Definition:** A technique based on block transformation coding to address difficulties of rendering large datasets on a GPU [S2].
**Context:** Proposed by Fout and Ma to enable real-time volume compression and decompression in GPU-based raycasting pipelines [S2].

## Background
Transform-based block coding has been used in volume visualization to address the difficulties of rendering large datasets on a GPU [S2]. Fout and Ma proposed a real-time volume compression and decompression technique based on block transformation coding to facilitate this [S2]. More recently, ZFP has been adopted as a de facto standard in the HPC community for fixed-rate compression of floating-point volume data [S1]. Recent work by Usher et al. ported ZFP to WebGPU for on-the-fly ISO surface extraction, and Dyken et al. used it for ray marching with progressive streaming [S1].

## Sources
[S1] GPU Volume Rendering with Hierarchical Compression Using VDB, https://arxiv.org/pdf/2504.04564, fetched 2026-10-26
[S2] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
