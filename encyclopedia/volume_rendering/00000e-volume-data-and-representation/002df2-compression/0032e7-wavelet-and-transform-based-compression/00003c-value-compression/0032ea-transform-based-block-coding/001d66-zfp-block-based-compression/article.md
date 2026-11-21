# ZFP Block-Based Compression

Also called: ZFP

## Overview
ZFP is a floating-point compression algorithm that encodes blocks of 4×4×4 values using fixed-rate, fixed-precision, or fixed-accuracy modes, serving as a de facto standard in the High-Performance Computing (HPC) community [S1, S2]. It distinguishes itself from sibling methods like JPEG2000 and TSDF Block Transform Coding by achieving exceptional rate-distortion performance and throughput (up to 2 GB/s) without introducing visual artifacts, while maintaining random access at block granularity [S2]. Unlike learned transform coding, ZFP relies on deterministic tensor-product transforms and bit-plane encoding, making it highly predictable and efficient for cache-based decompression [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume of floating-point values (typically double-precision, 64 bits/value) [S2].
*   **Output:** A compressed bitstream. In fixed-rate mode, the output size is strictly determined by the block size and target bit count [S2].

**Data Structures:**
*   **Block:** A 4×4×4 array of floating-point values [S1, S2].
*   **Transform Coefficients:** The result of applying a tensor-product transform to the block [S1].
*   **Bit Planes:** Binary representations of the transform coefficients, encoded sequentially [S1].

**Steps:**
1.  **Block Partitioning:** Partition the 3D input array into non-overlapping 4×4×4 blocks [S1, S2].
2.  **Normalization:** For each block, identify the largest exponent among all floating-point values. Factor out this common exponent to normalize the values [S1].
3.  **Fixed-Point Conversion:** Convert the normalized floating-point values into a two's complement fixed-point format [S1].
4.  **Transform:** Apply a spatial decorrelation transform to the block. This is implemented as a tensor product of 1D transforms along each axis. Supported transforms include the Discrete Wavelet Transform (DWT) and Discrete Cosine Transform (DCT) [S1].
5.  **Bit-Plane Encoding:** Encode the resulting transform coefficients bit by bit (per bit plane) [S1]. The specific entropy coding method (e.g., Huffman, Arithmetic) is not detailed in the provided sources, but the encoding is performed per bit plane [S1].

**Parameters:**
*   **Block Size:** Fixed at 4×4×4 [S1, S2].
*   **Precision Mode:**
    *   **Fixed-Rate:** Compresses each block to a fixed number of bits, enabling random access at block granularity [S2].
    *   **Fixed-Precision:** Fixes the number of uncompressed bits per value, resulting in bounded relative error [S2].
    *   **Fixed-Accuracy:** Guarantees each reconstructed value falls within a user-specified absolute error tolerance. This mode usually results in the smallest RMS error and highest Peak Signal-to-Noise Ratio (PSNR) for a given rate [S2].

**Formulas:**
*   **Accuracy vs. PSNR:** Every PSNR increase of 20 dB corresponds to an additional decimal digit of accuracy [S2].
*   **Error Reduction:** A 268 dB PSNR improvement represents an increase in accuracy (or reduction in RMS error) of more than $10^{12}$ times [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def zfp_compress_block(block_4x4x4_floats, mode, params):
    """
    Compress a single 4x4x4 block of floats using ZFP.
    
    Args:
        block_4x4x4_floats: 3D array of shape (4, 4, 4) containing float values.
        mode: 'fixed-rate', 'fixed-precision', or 'fixed-accuracy'.
        params: Dictionary containing mode-specific parameters (e.g., bits_per_block, 
                absolute_error_tolerance, bits_per_value).
    
    Returns:
        Compressed bitstream (bytes or bit array).
    """
    # Step 1: Find the maximum exponent in the block
    # Note: Floating point exponent extraction is hardware-specific.
    # We assume a helper function get_exponent(val) exists.
    max_exp = -infinity
    for x in range(4):
        for y in range(4):
            for z in range(4):
                exp = get_exponent(block_4x4x4_floats[x, y, z])
                if exp > max_exp:
                    max_exp = exp
    
    # Step 2: Normalize and convert to two's complement fixed-point
    # The normalization factor is 2^max_exp.
    # The fixed-point format must be chosen to accommodate the normalized range.
    # Let's assume a fixed-point format with F integer bits and F fractional bits.
    # The exact bit width for the fixed-point format depends on the mode and params.
    
    fixed_point_block = zeros((4, 4, 4), dtype=int)
    for x in range(4):
        for y in range(4):
            for z in range(4):
                val = block_4x4x4_floats[x, y, z]
                # Normalize: val / 2^max_exp
                normalized_val = val / (2 ** max_exp)
                # Convert to fixed-point. 
                # This step involves scaling and rounding to fit into the two's complement integer format.
                # The specific scaling factor and bit width are determined by the mode.
                fixed_point_block[x, y, z] = float_to_fixed_point(normalized_val, mode, params)
    
    # Step 3: Apply tensor-product transform
    # Apply 1D transform along X, then Y, then Z.
    # Supported transforms: DWT, DCT.
    transform_coeffs = apply_tensor_transform(fixed_point_block, transform_type='DWT') # Default to DWT
    
    # Step 4: Encode per bit plane
    # Extract bit planes from the transform coefficients.
    # Encode each bit plane. The specific entropy coding is not specified in sources.
    # We assume a generic bit-plane encoder.
    bitstream = encode_bit_planes(transform_coeffs, mode, params)
    
    return bitstream

def zfp_decompress_block(compressed_bitstream, mode, params):
    """
    Decompress a single 4x4x4 block.
    """
    # Step 1: Decode bit planes
    transform_coeffs = decode_bit_planes(compressed_bitstream, mode, params)
    
    # Step 2: Inverse tensor-product transform
    # Apply inverse 1D transform along Z, then Y, then X.
    fixed_point_block = apply_inverse_tensor_transform(transform_coeffs, transform_type='DWT')
    
    # Step 3: Convert back to floating-point
    # Reverse the normalization and fixed-point conversion.
    block_4x4x4_floats = zeros((4, 4, 4), dtype=float)
    for x in range(4):
        for y in range(4):
            for z in range(4):
                normalized_val = fixed_point_to_float(fixed_point_block[x, y, z], mode, params)
                # Re-apply the exponent
                block_4x4x4_floats[x, y, z] = normalized_val * (2 ** params['max_exp'])
                
    return block_4x4x4_floats
```

**Edge Cases and Pitfalls:**
*   **Cache Efficiency:** Decompression is only efficient when using caches. Incoherent memory access patterns, such as those in Monte Carlo rendering, require complex strategies like ray wavefronts, barrier synchronization, and sorting to amortize decompression costs [S1].
*   **Random Access:** Fixed-rate mode enables random access at block granularity, but this assumes the renderer can load and decompress individual blocks efficiently [S2].

## Subtopics

### Fixed-Rate Mode
Compresses each 4×4×4 block to a fixed number of bits. Best for applications requiring predictable memory usage and random access [S2].

### Fixed-Precision Mode
Fixes the number of bits per value, resulting in bounded relative error. Suitable when relative error is the primary concern [S2].

### Fixed-Accuracy Mode
Guarantees an absolute error tolerance. Best for applications requiring strict error bounds and highest PSNR [S2].

## Operating characteristics
*   **Throughput:** ZFP achieves up to 2 gigabytes/second throughput for compression and decompression [S2].
*   **Speed:** It compresses and decompresses data between 6 and 170 times faster than other compressors like ISABELA and fpzip [S2].
*   **Quality:** ZFP exhibits excellent rate-distortion performance with no visible artifacts, even at high compression ratios [S2].
*   **Memory Access:** Decompression efficiency is heavily dependent on cache coherence. Incoherent access patterns (e.g., ray tracing) can negate performance benefits without additional synchronization and sorting overhead [S1].
*   **Accuracy:** Fixed-accuracy mode can achieve a 268 dB PSNR improvement over ISABELA, representing a reduction in RMS error of more than 10 trillion times [S2].

## Background
ZFP was introduced by Lindstrom as a fast compressor and decompressor for floating-point data [S2]. It has become a de facto standard in the HPC community for compressing simulation data and volume rendering datasets [S1]. Recent work has ported ZFP to WebGPU for on-the-fly iso-surface extraction and progressive streaming [S1].

## Sources
[S1] GPU Volume Rendering with Hierarchical Compression Using VDB, https://arxiv.org/pdf/2504.04564, fetched 2026-10-26
[S2] zfp Compression Ratio and Quality - | Computing, https://computing.llnl.gov/projects/zfp/zfp-compression-ratio-and-quality, fetched 2026-10-29
