# Gaussian Splat Chunk Compression with Morton Reordering

Also called: Chunked Gaussian Splat Compression, Morton-Ordered Splat Textures

## Overview
This method reduces the storage size of trained 3D Gaussian Splatting models by reordering splats into spatially local chunks, normalizing data within each chunk to a 0..1 range, and storing the results in GPU textures using compression formats [S1, unofficial]. Unlike direct compression of raw data arrays, which suffers from poor compression ratios due to spatial discontinuity, this method aligns data with hardware compression block structures [S1, unofficial].

## How it works
**Inputs:** A list of Gaussian splats, where each splat contains position, scale, rotation, spherical harmonic coefficients, and opacity [S1, unofficial].
**Outputs:** Compressed GPU textures and chunk metadata (min/max values) for parameter reconstruction [S1, unofficial].

**Data Structures:**
1. **Chunk Metadata:** For each chunk, the minimum and maximum values for every data field (position, scale, color, SH) stored in full Float32 precision [S1, unofficial].
2. **Texture Layout:** Textures are organized such that one texel represents one splat [S1, unofficial]. Within a chunk of $N=256$ splats, the data is arranged in a $16 \times 16$ square layout [S1, unofficial].

**Parameters:**
* Chunk size $N$: 256 splats [S1, unofficial].
* Texture block dimension: $16 \times 16$ texels per chunk [S1, unofficial].

**Steps:**

1. **3D Morton Reordering:**
   Sort the splats in the data array based on their 3D positions using 3D Morton order [S1, unofficial]. The 3D Morton encoding for a vector $v = (x, y, z)$ of 21-bit integers is:
   $$ MortonEncode3(v) = (MortonPart1By2(v.z) \ll 2) \mid (MortonPart1By2(v.y) \ll 1) \mid MortonPart1By2(v.x) $$
   
   Where `MortonPart1By2` interleaves the bits of a 21-bit integer into a 64-bit integer:
   ```python
   def MortonPart1By2(x: int) -> int:
       x &= 0x1fffff
       x = (x ^ (x << 32)) & 0x1f00000000ffff
       x = (x ^ (x << 16)) & 0x1f0000ff0000ff
       x = (x ^ (x << 8)) & 0x100f00f00f00f00f
       x = (x ^ (x << 4)) & 0x10c30c30c30c30c3
       x = (x ^ (x << 2)) & 0x1249249249249249
       return x
   ```

2. **Chunking:**
   Group the reordered splats into contiguous chunks of size $N=256$ [S1, unofficial].

3. **Chunk Normalization:**
   For each chunk, compute the minimum and maximum values for all data fields across all splats in that chunk using full Float32 precision [S1, unofficial]. Normalize each splat's data values to the range $[0, 1]$ relative to the chunk's computed min/max [S1, unofficial]. Quaternions are encoded using the "smallest three" format, storing the three smallest components and the index of the largest component [S1, unofficial].

4. **2D Morton Reordering (Texture Layout):**
   Reorder the texture data within each chunk into a $16 \times 16$ square layout using 2D Morton order to align with BCn compression block structures [S1, unofficial]. The 2D Morton encoding for a coordinate $c = (x, y)$ within a $16 \times 16$ block is:
   ```python
   def EncodeMorton2D_16x16(x: int, y: int) -> int:
       t = ((y & 0xF) << 8) | (x & 0xF)
       t = (t ^ (t << 2)) & 0x3333
       t = (t ^ (t << 1)) & 0x5555
       return (t | (t >> 7)) & 0xFF
   ```

5. **Texture Compression:**
   Encode the normalized splat data into GPU textures using formats such as F32x4 (128-bit), F16x4 (64-bit), Norm10_2 (32-bit), Norm11 (32-bit), Norm8x4 (32-bit), Norm565 (16-bit), BC7 (8-bit), or BC1 (4-bit) [S1, unofficial]. Some formats like Norm10_2 require engine-specific emulation via Float32 textures and manual shader unpacking (e.g., Unity) [S1, unofficial].

**Edge Cases and Pitfalls:**
* **Compression Quality:** Storing data in $256 \times 1$ stripes causes poor compression quality with BCn formats because the compression is block-based ($4 \times 4$ texels) [S1, unofficial].
* **Large Chunk Bounds:** Morton curve flips can cause large chunk bounds, increasing variance and reducing normalization efficiency; partially filled chunks or Hilbert curves are suggested as mitigations [S1, unofficial].
* **Precision:** Normalization to 0..1 enables lower-precision formats, but format selection significantly impacts PSNR [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:** Not given in the sources [S1, unofficial].

**Measured Performance:**
* **Bike Scene:** Original 1.42GB. Very Low compression: 74MB (18.7x smaller, PSNR 24.02) [S1, unofficial].
* **Garden Scene:** Original 1.35GB. Medium compression: 267MB (5.2x smaller, PSNR 48.63) [S1, unofficial].
* **Comparison:** Direct BC7 compression on 256x1 stripe layout yields PSNR 21.71. Reordering into 16x16 blocks improves PSNR to 24.18 [S1, unofficial].

**Strengths and Limitations:**
* **Strengths:** Significant reduction in model size (up to ~19x) with acceptable quality loss at higher compression levels [S1, unofficial]. Leverages existing GPU texture hardware for decompression [S1, unofficial].
* **Limitations:** Requires preprocessing [S1, unofficial]. Lower quality presets (Very Low) may introduce visible artifacts [S1, unofficial]. Some texture formats require engine-specific workarounds [S1, unofficial].

## Background
This technique was introduced by Aras Pranckevičius to reduce the storage footprint of 3D Gaussian Splatting models, which typically store parameters in full Float32 precision [S1, unofficial]. The method relies on spatial locality via Morton ordering and chunk-wise normalization to make Gaussian data amenable to standard GPU texture compression techniques [S1, unofficial].

## Sources
[S1] Making Gaussian Splats smaller, https://aras-p.info/blog/2023/09/13/Making-Gaussian-Splats-smaller/, fetched 2026-10-26 (unofficial)
