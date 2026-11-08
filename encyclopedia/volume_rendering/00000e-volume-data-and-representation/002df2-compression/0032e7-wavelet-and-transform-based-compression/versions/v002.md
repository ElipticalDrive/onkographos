# Wavelet and Transform-based Compression

Also called: Transform-based Volume Compression, Frequency Domain Compression

## Overview
Wavelet and Transform-based Compression reduces volumetric data size by projecting the signal into a frequency or residual domain where energy is compacted, allowing for efficient quantization and entropy coding [S1]. It solves the problem of high memory bandwidth and storage requirements for large 3D datasets by exploiting spatial correlation and smoothness [S1]. What sets it apart from [Value compression](6-value-compression.md), which operates directly on amplitude statistics, and [Sparse structures](5-sparse-structures.md), which address geometric sparsity, is its reliance on linear basis functions (wavelets, pyramids) to decorrelate data, enabling multi-resolution access and superior handling of smooth gradients [S1][S5].

## Shared principles
All methods in this family share the mechanism of decomposing the input volume $V$ into a set of coefficients $C$ using a linear transform, followed by encoding and optional decoding for rendering.

1.  **Transform:** The volume is decomposed using a separable 1D filter bank applied along each axis. For a 3D volume, this results in subbands representing different frequency components. In wavelet transforms, these are typically approximation (low-pass) and detail (high-pass) coefficients. In Laplacian pyramids, they are detail residuals between levels of a Gaussian pyramid.
2.  **Hierarchy:** The decomposition is recursive. The approximation subband from level $j$ is further decomposed at level $j+1$, creating a tree-like structure of coefficients. This allows for multi-resolution access, where coarse data is available at the root and fine details are deeper in the tree.
3.  **Quantization and Entropy Encoding:** Coefficients are quantized (lossy) or kept precise (lossless). Entropy coding (e.g., Huffman, Arithmetic, Run-Length) is applied to the quantized or raw coefficients to produce the compressed bitstream.
4.  **Reconstruction:** To render, the necessary coefficients are decoded and the inverse transform is applied. This can be done globally (full reconstruction) or locally (block-wise or point-wise) to support adaptive resolution rendering.

## Subtopics

### [Value compression](6-value-compression.md)
Reduces storage by encoding voxel values directly, distinct from spatial compression. It manipulates the amplitude domain to minimize bit-depth.

### [Wavelet and Pyramid Transforms](3856-wavelet-and-pyramid-transforms.md)
Compresses data by decomposing $V$ into coefficients $C$ using linear transforms (wavelets/pyramids), supporting multi-resolution access via recursive hierarchy.

### [Blockwise Wavelet Compression](6100-blockwise-wavelet-compression.md)
Decomposes 3D data into small, equal-sized blocks to enable independent compression and parallel processing, localizing the transform.

## Background
The application of wavelet transforms to volumetric data originated in the mid-1990s as a generalization of 2D image compression techniques. Early work by researchers such as Sweldens and Pierrakos explored the use of lifting schemes and separable 3D wavelets for medical imaging [S1]. The method became standard in scientific visualization for its ability to provide lossless compression with high ratios for smooth data, and lossy compression with minimal visual artifacts for ray-marched volumes. It is widely used in medical imaging (DICOM), climate data archives, and real-time volume rendering systems that require level-of-detail (LOD) streaming.

## Sources
[S1] Blockwise Wavelet Compression notes, URL, date fetched
