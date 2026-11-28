# Texture Format Compression

Also called: Block Compression, BCn, DXT, S3TC

## Overview
Texture Format Compression refers to the use of standardized, block-based compression algorithms originally designed for 2D image data to compress volumetric or texture data [S1, unofficial][S2, book][S3, unofficial]. It solves the problem of high memory bandwidth and storage requirements by compressing data into fixed-size bitstreams [S1, unofficial][S2, book]. Not given in the sources.

## Shared principles
The family relies on block-based compression formats (e.g., BC4, BC5, BC6H, DXT1) to compress texture-like data into fixed-size bitstreams [S1, unofficial][S2, book][S3, unofficial][S4, unofficial]. Not given in the sources.

## Subtopics

### [BC5 Compression](5013-bc5-compression.md)
BC5 is a block compression format for 2-channel (RG) textures, compressing 4x4 blocks (32 bytes) into 16 bytes. It supports two variants: BC5_UNORM (outputs [0, 1] floats) and BC5_SNORM (outputs [-1, 1] floats). It is the 2-channel extension of BC4, which handles single-channel (R) data.

### Distance Field Compression
#### definition
- [S1, unofficial] Compression of the distance field texture (e.g., using BCn) to save memory/bandwidth, ensuring compressed values remain lower than originals to avoid overshooting.
#### pitfall
- [S1, unofficial] Compressed values must stay lower than the original uncompressed values to avoid overshooting the cloud boundary.

### Quantized Texture Representation
#### definition
- [S2, book] Re-encoding NanoVDB float32 density data into compressed texture formats to reduce memory footprint.
#### parameter
- [S2, book] Selected format: BC4 (4 bits per texel).
#### number
- [S2, book] BC4 compression results in a PSNR of 54.34 dB and a FLIP error of 0.00738 compared to the baseline.

### HDR Image Compression via Log-RGB Tree
#### definition
- [S3, unofficial] A scheme to compress HDR images by capturing high-dynamic range variations using an aggressively compressed tree on log(RGB) data, and encoding the remaining low-dynamic range detail using standard DXT1 compression.
#### step
- [S3, unofficial] Apply tree compression to the log(RGB) image to capture HDR variation.
- [S3, unofficial] Quantize the channels of the detail image separately to 8 bits.
- [S3, unofficial] Compress the quantized detail image using ordinary DXT1 compression.
#### parameter
- [S3, unofficial] Bit rate for the tree-compressed log(RGB) component: 1 bit/pixel.
- [S3, unofficial] Bit rate for the DXT1 detail component: 4 bits/pixel.
- [S3, unofficial] Overall representation bit rate: 5 bits/pixel.
#### pitfall
- [S3, unofficial] Encoding all 3 color channels (rather than just luminance) reduces subsequent hue quantization artifacts.
#### comparison
- [S3, unofficial] Compares favorably with Munkberg et al [MCHA06] which uses 8 bits/pixel.

### BC6H Block Compression (P2 Mode)
#### definition
- [S4, unofficial] A block compression method using two endpoints and 3-bit indices (8 levels) with predefined patterns, offering higher quality at slower performance.
#### step
- [S4, unofficial] Evaluate all 32 patterns using EvaluateP2Pattern to find the one with minimum squared distance from lines defined by endpoints.
- [S4, unofficial] For the selected pattern, split texels into two groups based on paletteID (0 or non-0) and compute separate bounding boxes for each group.
- [S4, unofficial] Compute direction vectors for each group: p0BlockDir = normalize(p0BlockMax - p0BlockMin); p1BlockDir = normalize(p1BlockMax - p1BlockMin).
- [S4, unofficial] Calculate total squared distance from texels to their respective line segments for error evaluation.

## Background
Block compression formats are used in volume rendering to compress density fields [S2, book] and distance fields [S1, unofficial]. The BC5 format extends block compression to dual-channel data, commonly used for normal maps or displacement fields [S1, unofficial]. In volume rendering, these formats are adapted for compressing density fields [S2, book] and distance fields [S1, unofficial]. The use of BC4 for NanoVDB data demonstrates its applicability to scientific volume data, achieving significant compression ratios with minimal perceptual error [S2, book]. HDR compression techniques have also adapted block-based principles by combining tree structures with DXT1 for detail layers [S3, unofficial]. Not given in the sources.

## Sources
[S1] Volumetric Cloud Rendering - Jack Tollenaar, https://www.jacktollenaar.top/articles/clouds.html, fetched 2026-10-29 (unofficial)
[S2] Efficient Unbiased Volume Path Tracing on the GPU, https://doi.org/10.1007/978-1-4842-7185-8_43, fetched 2026-10-24 (book)
[S3] Compressed Random-Access Trees for Spatially Coherent Data, https://doi.org/10.2312/egwr/egsr07/339-349, fetched 2026-10-26 (unofficial)
[S4] Real-time BC6H compression on GPU, https://github.com/knarkowicz/GPURealTimeBC6H, fetched 2026-10-27 (unofficial)
