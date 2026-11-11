# Wavelet Transform and Decomposition
Also called: Wavelet-based Volume Compression, Multiresolution Volume Decomposition

## Overview
This family encompasses methods that compress volumetric scalar fields by decomposing the input volume $V$ into a set of coefficients $C$ using linear transforms, followed by encoding and optional decoding for rendering. It differs from its siblings by relying on frequency-domain or multi-resolution basis functions (wavelets, Laplacian pyramids) rather than spatial prediction, fractal subdivision, or block-based vector quantization alone.

## Shared principles
All methods in this family share the mechanism of decomposing the input volume $V$ into a set of coefficients $C$ using a linear transform, followed by encoding and optional decoding for rendering.

1.  **Transform:** The volume is decomposed using a separable 1D filter bank applied along each axis. For a 3D volume, this results in subbands representing different frequency components. In wavelet transforms, these are typically approximation (low-pass) and detail (high-pass) coefficients. In Laplacian pyramids, they are detail residuals between levels of a Gaussian pyramid.
2.  **Hierarchy:** The decomposition is recursive. The approximation subband from level $j$ is further decomposed at level $j+1$, creating a tree-like structure of coefficients. This allows for multi-resolution access, where coarse data is available at the root and fine details are deeper in the tree.
3.  **Quantization and Entropy Encoding:** Coefficients are quantized (lossy) or kept precise (lossless). Entropy coding (e.g., Huffman, Arithmetic, Run-Length) is applied to the quantized or raw coefficients to produce the compressed bitstream.
4.  **Reconstruction:** To render, the necessary coefficients are decoded and the inverse transform is applied. This can be done globally (full reconstruction) or locally (block-wise or point-wise) to support adaptive resolution rendering.

## Subtopics

### [Triadic frequency band decomposition](828-triadic-frequency-band-decomposition.md)
A lossy scheme partitioning data into $4^3$ blocks and decomposing them into three frequency bands (high, mid, mean) encoded via vector quantization and hardware texture lookups. Chose for hardware-accelerated decoding with fixed block structures.

### [Laplacian Pyramid](2250-laplacian-pyramid.md)
A multiresolution technique using non-orthogonal, redundant filters to create detail residuals and a Gaussian root. Chose for progressive rendering and perfect reconstruction despite oversampling.

### [Wavelet Kernel Selection](7035-wavelet-kernel-selection.md)
The process of choosing filter coefficients for lifting schemes to balance complexity, reversibility, and energy compaction. Chose when specific wavelet properties (e.g., integer-preserving for lossless) are required.

## Background
The application of wavelet transforms and multiresolution analysis to volume data compression emerged in the late 1990s and early 2000s as a response to the need for efficient storage and progressive transmission of large volumetric datasets [G]. The Laplacian Pyramid, originally developed for image compression by Burt and Adelson, was adapted for volumes to handle inter-voxel correlation [S1, unofficial]. Wavelet-based methods, leveraging the energy compaction properties of orthogonal and biorthogonal wavelets, became standard for lossy compression, while lifting schemes were introduced to enable lossless compression with integer-to-integer transforms [S1].

## Sources
[S1] Unofficial notes on Triadic, Laplacian, and Wavelet Kernel details.
