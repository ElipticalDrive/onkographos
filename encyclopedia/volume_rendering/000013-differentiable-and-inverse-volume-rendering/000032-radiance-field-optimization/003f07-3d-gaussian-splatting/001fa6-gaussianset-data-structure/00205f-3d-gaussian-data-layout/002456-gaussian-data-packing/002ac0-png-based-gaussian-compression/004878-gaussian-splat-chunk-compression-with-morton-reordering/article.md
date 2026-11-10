# Gaussian Splat Chunk Compression with Morton Reordering

Also called: Chunked Gaussian Splat Compression, Morton-Ordered Splat Textures

## Overview
This method reduces the storage size of trained 3D Gaussian Splatting models by reordering splats into spatially local chunks, normalizing data within each chunk to a 0..1 range, and storing the results in GPU textures using compression formats [S1, unofficial]. It builds on [PNG-based Gaussian Compression](4834-png-based-gaussian-compression.md) by introducing spatial locality via Morton ordering to improve the efficiency of block-based texture compression (BCn) [S1, unofficial]. Unlike direct compression of raw data arrays, which suffers from poor compression ratios due to spatial discontinuity, this method aligns data with hardware compression block structures [S1, unofficial].

## How it works
**Inputs:** A list of $N_{total}$ Gaussian splats, where each splat contains position ($\mathbf{p}$), scale ($\mathbf{s}$), rotation ($\mathbf{q}$), spherical harmonic coefficients ($\mathbf{c}$), and opacity ($\alpha$).
**Outputs:** A set of compressed GPU textures and metadata (chunk min/max values) allowing reconstruction of splat parameters.

**Data Structures:**
1.  **Chunk Metadata:** For each chunk, store the minimum and maximum values for every data field (position, scale, color, SH) in full Float32 precision.
2.  **Texture Layout:** Textures are organized such that one texel represents one splat. Within a chunk of $N=256$ splats, the data is arranged in a $16 \times 16$ square layout.

**Parameters:**
*   Chunk size $N$: 256 splats [S1, unofficial].
*   Texture block dimension: $16 \times 16$ texels per chunk [S1, unofficial].

**Steps:**

1.  **3D Morton Reordering:**
    Sort the splats in the data array based on their 3D positions using 3D Morton order. This ensures that spatially nearby splats are adjacent in the array.
    
    The 3D Morton encoding for a vector $v = (x, y, z)$ of 21-bit integers is:
    $$ MortonEncode3(v) = (MortonPart1By2(v.z) \ll 2) \mid (MortonPart1By2(v.y) \ll 1) \mid MortonPart1By2(v.x) $$
    
    Where `MortonPart1By2` interleaves the bits of a 21-bit integer into a 64-bit integer:
    ```python
    def MortonPart1By2(x: int) -> int:
        # x is assumed to be a 21-bit integer (0 <= x < 2^21)
        x &= 0x1fffff
        x = (x ^ (x << 32)) & 0x1f00000000ffff
        x = (x ^ (x << 16)) & 0x1f0000ff0000ff
        x = (x ^ (x << 8)) & 0x100f00f00f00f00f
        x = (x ^ (x << 4)) & 0x10c30c30c30c30c3
        x = (x ^ (x << 2)) & 0x1249249249249249
        return x
    ```

2.  **Chunking:**
    Group the reordered splats into contiguous chunks of size $N=256$.

3.  **Chunk Normalization:**
    For each chunk, compute the global minimum and maximum values for all data fields across all splats in that chunk using full Float32 precision.
    
    Normalize each splat's data values to the range $[0, 1]$ relative to the chunk's computed min/max:
    $$ v_{norm} = \frac{v - v_{min}}{v_{max} - v_{min}} $$
    
    *Rotation Encoding:* Quaternions are encoded using the "smallest three" format. Store the three smallest components of the quaternion and the index of the largest component. This allows reconstruction of the full quaternion (up to sign ambiguity, which is resolved by the sign of the largest component or standard quaternion normalization) [S1, unofficial].

4.  **2D Morton Reordering (Texture Layout):**
    Reorder the texture data within each chunk into a $16 \times 16$ square layout using 2D Morton order. This aligns the data with the $4 \times 4$ texel blocks used by GPU texture compression formats (e.g., BC7, BC1) [S1, unofficial].
    
    The 2D Morton encoding for a coordinate $c = (x, y)$ within a $16 \times 16$ block is:
    ```python
    def EncodeMorton2D_16x16(x: int, y: int) -> int:
        # x, y are 4-bit integers (0 <= x, y < 16)
        t = ((y & 0xF) << 8) | (x & 0xF)
        t = (t ^ (t << 2)) & 0x3333
        t = (t ^ (t << 1)) & 0x5555
        return (t | (t >> 7)) & 0xFF
    ```

5.  **Texture Compression:**
    Encode the normalized splat data into GPU textures. Select a texture format based on the desired quality/compression trade-off. Common formats include:
    *   **F32x4:** 128-bit (4x Float32)
    *   **F16x4:** 64-bit (4x Float16)
    *   **Norm10_2:** 32-bit (unsigned normalized 10.10.10.2)
    *   **Norm11:** 32-bit (unsigned normalized 11.10.11)
    *   **Norm8x4:** 32-bit (4x unsigned normalized byte)
    *   **Norm565:** 16-bit (unsigned normalized 5.6.5)
    *   **BC7:** 8-bit (GPU texture compression)
    *   **BC1:** 4-bit (GPU texture compression)
    
    Note: Some formats like Norm10_2 may not be directly supported by all engines (e.g., Unity), requiring emulation via Float32 textures and manual unpacking in shaders [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Compression Quality:** Storing data in $256 \times 1$ stripes causes poor compression quality with BCn formats because the compression is block-based ($4 \times 4$ texels). Reordering into $16 \times 16$ blocks is critical [S1, unofficial].
*   **Large Chunk Bounds:** If Morton curve flips cause large chunk bounds, variance within the chunk increases, reducing normalization efficiency. Future work suggests partially filled chunks or Hilbert curves [S1, unofficial].
*   **Precision:** Normalization to 0..1 allows the use of lower-precision formats, but the choice of format affects PSNR significantly [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Preprocessing:** Sorting splats by Morton code is $O(N \log N)$. Chunk normalization is $O(N)$. Texture layout reordering is $O(N)$.
*   **Memory:** The compressed size depends on the texture format. For example, the "Very Low" compression preset achieves ~18.7x smaller size than the original Float32 data [S1, unofficial].

**Measured Performance:**
*   **Bike Scene:** Original 1.42GB. Very Low compression: 74MB (18.7x smaller, PSNR 24.02) [S1, unofficial].
*   **Garden Scene:** Original 1.35GB. Medium compression: 267MB (5.2x smaller, PSNR 48.63) [S1, unofficial].
*   **Comparison:** Direct BC7 compression on 256x1 stripe layout yields PSNR 21.71. Reordering into 16x16 blocks improves PSNR to 24.18 [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Significant reduction in model size (up to ~19x) with acceptable quality loss at higher compression levels. Leverages existing GPU texture hardware for decompression.
*   **Limitations:** Requires preprocessing step. Lower quality presets (Very Low) may introduce visible artifacts. Some texture formats require engine-specific workarounds [S1, unofficial].

## Background
This technique was introduced by Aras Pranckevičius in a blog post detailing methods to make Gaussian Splats smaller [S1, unofficial]. It addresses the large storage footprint of 3D Gaussian Splatting models, which typically store parameters in full Float32 precision. The method builds on the observation that spatial locality (via Morton ordering) and chunk-wise normalization can make Gaussian data amenable to standard GPU texture compression techniques [S1, unofficial].

## Sources
[S1] Making Gaussian Splats smaller, https://aras-p.info/blog/2023/09/13/Making-Gaussian-Splats-smaller/, fetched 2026-10-26 (unofficial)
