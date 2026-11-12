# Hardware and Format Specific Compression
## Overview
This family covers compression methods tailored to specific hardware formats or data representations rather than general statistical redundancy [G]. It addresses memory bandwidth and storage inefficiency by mapping volumetric data into formats natively supported by GPU architectures or specialized hardware accelerators [G].
## Shared principles
The children in this family share no common algorithmic mechanism beyond the general principles of [Compression](4-compression.md) [G].
## Subtopics

### [16-bit Floating Point Format for Spectra](1163-16-bit-floating-point-format-for-spectra.md)
Defines a custom 16-bit floating point format for frequency domain spectral coefficients, reducing memory cost from 8× to 2× the original spatial volume size while maintaining acceptable precision for rendering [1163-16-bit-floating-point-format-for-spectra.md]. It is chosen when rendering requires frequency-domain transforms (e.g., Hartley) and the wide dynamic range of coefficients exceeds integer precision limits.

### [Color Packing (RGBA)](5142-color-packing-rgba.md)
Maps 4-component floating-point color vectors into a compact 4-byte unsigned integer representation with 8-bit precision per channel [5142-color-packing-rgba.md]. It differs from spectral compression by operating on spatial color data with fixed-point integer storage, and is chosen when reducing memory bandwidth for color/alpha channels is prioritized over floating-point precision.

### [ELSCINT1/PMSCT_RLE1 Delta Decoding](11897-elscint1-pmsct-rle1-delta-decoding.md)
Decodes a custom RLE + Delta compressed byte stream used by ELSCINT1/PMSCT medical imaging devices into 16-bit unsigned integers [11897-elscint1-pmsct-rle1-delta-decoding.md]. It differs from general codecs by using specific escape markers (`0xa5` for RLE, `0x5a` for delta) and a two-pass decoding process, and is chosen when interfacing with legacy medical imaging hardware formats.

### RGB Channel Packing (VTC)
Packs three time steps of volumetric data into the Red, Green, and Blue channels of a 3D texture, then applies standard Video Texture Compression (VTC) algorithms designed for 3D-RGB data [S1]. It is chosen for time-varying volumes when leveraging existing hardware or software VTC decoders for temporal compression is advantageous.

## Background
The family originates from research on compressed GPU-based direct volume rendering [S1]. Early work by Fout et al. [FAM∗ 05] proposed methods for multivariate time-varying volumes that group voxels in temporal or spatial domains and apply vector quantization algorithms to corresponding blocks [S1]. Nagayasu et al. [NIH08] introduced packing three time steps into RGB channels followed by standard VTC compression [S1]. Ko et al. [KLW∗ 08] classified time steps as intra-coded or predictive frames, encoding all frames with Haar wavelets and predictive frames with delta encoding [S1]. Mensman et al. [MRH10] performed block-based delta encoding with respect to previous time steps, quantized blocks with variable bit counts, and encoded them with LZO [S1]. She et al. [SBN11] further developed delta encoding techniques for volume data [S1].
## Sources
[S1] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
