# Wavelet Transform and Decomposition
Also called: Wavelet and Pyramid Transforms, Multiresolution Analysis

## Overview
This family encompasses methods that decompose a volumetric scalar field $V$ into a set of coefficients $C$ using linear transforms, enabling compression and multi-resolution rendering. It is distinguished from sibling methods like [Predictive and Differential Coding](4393-predictive-and-differential-coding.md) by its reliance on frequency-domain or hierarchical basis decompositions rather than spatial prediction errors, and from [Quantization](5935-quantization.md) by its explicit separation of signal into approximation and detail subbands.

## Shared principles
All methods in this family share the mechanism of decomposing the input volume $V$ into a set of coefficients $C$ using a linear transform, followed by encoding and optional decoding for rendering.

1.  **Transform:** The volume is decomposed using a separable 1D filter bank applied along each axis. For a 3D volume, this results in subbands representing different frequency components. In wavelet transforms, these are typically approximation (low-pass) and detail (high-pass) coefficients. In Laplacian pyramids, they are detail residuals between levels of a Gaussian pyramid [see [Wavelet and Pyramid Transforms](3856-wavelet-and-pyramid-transforms.md)].
2.  **Hierarchy:** The decomposition is recursive. The approximation subband from level $j$ is further decomposed at level $j+1$, creating a tree-like structure of coefficients. This allows for multi-resolution access, where coarse data is available at the root and fine details are deeper in the tree [see [Value compression](6-value-compression.md)].
3.  **Quantization and Entropy Encoding:** Coefficients are quantized (lossy) or kept precise (lossless). Entropy coding (e.g., Huffman, Arithmetic, Run-Length) is applied to the quantized or raw coefficients to produce the compressed bitstream [see [Value compression](6-value-compression.md)].
4.  **Reconstruction:** To render, the necessary coefficients are decoded and the inverse transform is applied. This can be done globally (full reconstruction) or locally (block-wise or point-wise) to support adaptive resolution rendering [see [Value compression](6-value-compression.md)].

## Subtopics

### [Triadic frequency band decomposition](828-triadic-frequency-band-decomposition.md)
A lossy method partitioning data into $4^3$ blocks and decomposing them into three frequency bands (high, mid, mean) using fixed hardware-friendly downsampling. It differs from siblings by mapping directly to dependent texture fetches for on-chip decoding, whereas [Laplacian Pyramid](2250-laplacian-pyramid.md) uses Gaussian pyramids and [Wavelet Kernel Selection](7035-wavelet-kernel-selection.md) focuses on optimizing basis functions [G].

### [Laplacian Pyramid](2250-laplacian-pyramid.md)
A multiresolution technique representing data as a low-resolution base plus full-resolution difference signals (residuals) derived via Gaussian low-pass filtering and upsampling. It sets itself apart from orthogonal wavelets by using a non-orthogonal, redundant representation that guarantees perfect reconstruction through simple addition.

### [Wavelet Kernel Selection](7035-wavelet-kernel-selection.md)
Determines specific filter coefficients (e.g., Haar, 5x3, 9x7) for the Discrete Wavelet Transform. It solves the trade-off between lossless reversibility and lossy energy compaction, relying on biorthogonal or orthogonal filter banks with specific vanishing moments rather than fixed polynomial or pyramid-based splitting.

### [Q2 Finite Element Compression](17905-q2-finite-element-compression.md)
A multiresolution scheme using a hierarchical basis of quadratic polynomials ($P_2$) to decompose signals into coarse and detail coefficients. It differs from siblings by using a continuous piecewise quadratic basis, allowing for linear complexity and sparse storage without explicit quantization steps.

## Background
The theoretical foundation of wavelet transforms was established in the late 1980s and early 1990s, with key contributions from Mallat  and Meyer . The application of wavelets to volume rendering and compression emerged in the late 1990s, notably with the work of Westermann  and others, leveraging the multi-resolution properties for adaptive rendering. Laplacian pyramids, introduced by Burt and Adelson , predate wavelets but share the hierarchical decomposition philosophy. Q2 finite element compression draws from finite element analysis literature .

## Sources
[S1] Unofficial notes on Triadic, Laplacian, and Q2 methods.
[S2] S. Mallat, "A theory for multiresolution signal decomposition: the wavelet representation," IEEE Transactions on Pattern Analysis and Machine Intelligence, 1989.
[S3] Y. Meyer, "Wavelets and Operators," Cambridge University Press, 1993.
[S4] R. Westermann, "Efficient Volume Rendering of Data with Large Dynamic Range Using a Wavelet-Based Transfer Function," Eurographics, 1999.
[S5] P. J. Burt and E. H. Adelson, "The Laplacian Pyramid as a Compact Image Code," IEEE Transactions on Communications, 1983.
