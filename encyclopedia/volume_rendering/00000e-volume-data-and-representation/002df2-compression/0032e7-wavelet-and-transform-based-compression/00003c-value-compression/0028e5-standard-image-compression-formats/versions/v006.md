# Standard Image Compression Formats

Also called: Block Compression, Texture Compression

## Overview
Standard Image Compression Formats refer to a family of lossy compression codecs, primarily designed for 2D texture atlases, that are adapted for volume rendering by treating volumetric data or splat attributes as dense 2D surfaces or 3D blocks [S1, unofficial]. This family solves the problem of high memory bandwidth and storage requirements for large-scale volumetric or neural rendering datasets by leveraging hardware-accelerated decompression [S1, unofficial]. It sets itself apart from general-purpose volume compression methods (like [Value compression](6-value-compression.md)) by relying on fixed-rate, block-based spatial locality rather than semantic or topological volume structures, enabling direct hardware rasterization without full in-memory decompression [S1, unofficial].

## Shared principles
All methods in this family share the mechanism of decomposing the input data into fixed-size spatial blocks, encoding each block independently using endpoint and index quantization, and providing hardware support for on-the-fly reconstruction during rendering [S1, unofficial].
1. **Block Partitioning:** The input data is divided into non-overlapping blocks of fixed dimensions [S1, unofficial]. For 2D formats like BC7, the standard block size is $4 \times 4$ texels [S1, unofficial].
2. **Endpoint and Index Encoding:** Each block is compressed by storing a small set of "endpoint" values (base colors or scalar values) and a set of "index" values (weights) that define a linear interpolation grid [S1, unofficial].
3. **Fixed-Rate Compression:** The compressed size is deterministic based on the block dimensions and color format (e.g., RGB vs. RGBA) [S1, unofficial]. This allows the renderer to calculate memory offsets directly from coordinates without parsing a variable-length bitstream [S1, unofficial].
4. **Hardware Acceleration:** The primary advantage is that modern GPUs can decompress these blocks directly from compressed memory into registers during the fragment/pixel shader stage, maintaining single-fetch performance [S1, unofficial].

## Subtopics

### [Variable-Block Color Compression](1878-variable-block-color-compression.md)
Variable-Block Color Compression is a lossy compression method for sparse 3D color data that encodes color values using variable-length blocks rather than fixed-size grids. It solves the problem of high memory overhead in sparse voxel representations by grouping contiguous voxels with similar colors into single blocks, each defined by two base colors and a weight. It sets itself apart from sibling methods like [ASTC](5729-astc.md) by using a greedy merging algorithm to determine block boundaries dynamically based on an error threshold, rather than relying on a fixed partitioning scheme, and by storing headers in a fixed-size array to enable efficient binary search decoding.

### [ASTC](5729-astc.md)
ASTC is a flexible, hardware-supported, fixed-rate lossy compression method that uses configurable block-size configurations to cover a wide range of bit rates. It adapts the principles of Block Truncated Coding (BTC) and S3TC to support hardware-accelerated random access, allowing rendering to reconstruct voxel values in real-time from compressed memory without decompressing the entire volume. Unlike its sibling [Variable-Block Color Compression](1878-variable-block-color-compression.md), which focuses on variable block splitting, ASTC provides a standardized set of fixed block dimensions with scalable bit-depths, making it suitable for both 2D textures and 3D volumetric data.

### Block-Dictionary Compression
#### definition
- [S1, unofficial] Optional compression for distribution where the BC7 stream is compressed with a block dictionary.
#### step
- [S1, unofficial] Cut atlas into 4 × 4 texel blocks.
- [S1, unofficial] K-means clustering builds a codebook of K = 65,536 block centroids.
- [S1, unofficial] Each centroid is encoded as one 16-byte BC7 block.
- [S1, unofficial] Every atlas block stores a 16-bit codebook index.
- [S1, unofficial] Load time: Indices are gathered back into a dense BC7 stream, so the renderer is unchanged and keeps a single hardware bilinear fetch per fragment.
#### number
- [S1, unofficial] Codebook size: 1 MB. Index size: 2 bytes per block. Reduction: 7.0× of the atlas payload (173 MB to 24.7 MB on average for Mip-NeRF 360).
- [S1, unofficial] Complete web bundle size: 38.4 MB (dictionary atlas with 8-bit surfels (FP16 positions) and cameras).
#### comparison
- [S1, unofficial] The dictionary reduces download size, while GPU memory and frame time are those of the raw BC7 atlas.

### BC7 Quantization with Empirical Range
#### definition
- [S1, unofficial] Compression of the baked atlas using BC7 block compression. The residual range is estimated empirically to avoid clipping heavy tails, and the scale is snapped to align zero residual with an integer quantization level [S1, unofficial].
#### step
- [S1, unofficial] 1. Estimate residual range as empirical [min, max] with a 2% margin over ~1.3 million samples (64 random UV positions on 20,000 surfels) [S1, unofficial]. 2. Quantize uniformly with a single global (offset, scale) pair [S1, unofficial]. 3. Snap scale so residual zero coincides with an integer quantization level [S1, unofficial]. 4. Pad data to RGBA to match BC7 4x4 block layout (16 bytes per block, 1 byte per texel) [S1, unofficial].
#### pitfall
- [S1, unofficial] Using range estimates of the form $\mu \pm k\sigma$ clips heavy tails, dimming bright peaks uniformly. Using the full empirical range avoids this [S1, unofficial].
- [S1, unofficial] If the scale is not snapped to align zero residual with an integer level, halfstep rounding bias propagates through the outer ReLU of Eq. 2, systematically dimming pixels near the activation threshold [S1, unofficial].
#### number
- [S1, unofficial] BC7 RGBA storage uses 1 byte/texel (~173 MB mean atlas size) with ±0.01 dB PSNR change compared to neural FP16 RGB (6 bytes/texel, ~1039 MB) [S1, unofficial].

### Complex Hologram Compression & Evaluation Pipeline
#### definition
- [S2] Compression architecture for complex-valued hologram data, splitting the matrix into real and imaginary components for standard codec encoding.
#### step
- [S2] Split the complex matrix into two distinct real-valued matrices: Real (Re) and Imaginary (Im).
- [S2] Quantize the Re and Im matrices to 16 bits per sample (bps).
- [S2] Encode the quantized Re and Im components separately using a standard image/video codec.
- [S2] Merge the decoded Re and Im components back into a single complex matrix.
- [S2] Reconstruct the wavefield from both the decoded complex matrix and the original raw complex matrix.
- [S2] Evaluate quality in the hologram domain by comparing pre- and post-compression Re/Im components using PSNR and SSIM.
- [S2] Evaluate quality in the reconstructed domain by comparing the reconstructed images from decoded and raw data using PSNR and SSIM.
#### parameter
- [S2] 16 bps: fixed quantization depth for real and imaginary components to match standard codec integer input requirements.
#### layout
- [S2] Input: Complex raw data, a 2D matrix of complex values in algebraic form (real and imaginary parts).
#### comparison
- [S2] Algebraic form is preferred over polar form for compression because prior state-of-the-art work demonstrates better compression performance.

## Background
The foundation of this family lies in the S3 Texture Compression (S3TC) and its successors, developed by S3 Graphics in the late 1990s and early 2000s for real-time 3D graphics [G]. BC7 (Block Compression 7) was introduced as part of the OpenGL Extension GL_EXT_texture_compression_bc7, offering high-quality compression for complex color data [G]. ASTC (Adaptive Scalable Texture Compression) was developed by ARM as a more flexible alternative, supporting variable block sizes and bit rates [G]. In the context of volume rendering and neural rendering, these formats have been adapted to compress texture atlases that store splat attributes (positions, colors, opacities) or baked neural field representations, leveraging the existing GPU hardware pipeline to avoid the memory bottleneck of storing dense FP16/FP32 volumes [S1, unofficial].

## Sources
[S1] Bake It Till You Make It: Ultrafast Spatial Texture-Atlas Splatting, https://doi.org/10.48550/arXiv.2607.13808, fetched 2026-10-29 (unofficial)
[S2] Recent Advances in the Processing and Rendering Algorithms for Computer-Generated Holography, https://doi.org/10.3390/electronics8050556, fetched 2026-10-26
