# Hierarchical Vector Quantization

Also called: HVQ Compression, HVQ rendering, Hierarchical Decomposition, Hierarchical Vector Quantization (VQ) for Time-Varying Scalar Fields, Hierarchical Vector Quantization for Volume Rendering, VQ compression

## Overview
Hierarchical Vector Quantization (HVQ) is a lossy compression algorithm for volumetric data that achieves high compression ratios (typically ~20:1) with minimal artifacts by representing volume blocks as indices into learned codebooks [S2, S3, unofficial]. It is distinguished from sibling methods like [Wavelet Compression](52-wavelet-compression.md) and [S3 Texture Compression](958-s3-texture-compression.md) by its ability to decode and render compressed data directly on the GPU during the rendering pass, thereby minimizing CPU-to-GPU data transfer and circumventing limited texture memory [S2, S3, unofficial]. Unlike [Vector Quantization](907-vector-quantization.md) which uses a single dictionary, HVQ employs multiple resolution levels (codebooks) to capture both coarse and fine details, allowing for efficient handling of large-scale and time-varying data [S3, unofficial][S4].

## How it works
**Inputs:** Static or time-varying volumetric scalar data $S_t$ at time $t$.
**Outputs:** Compressed file containing headers, two codebooks (A and B), and RGB index triplets for rendering.

**Data Structures:**
1.  **Codebook A:** 256 entries, each 64-dimensional (stores detail level 0 vectors). Size: $256 \times 64$ bytes (assuming 8-bit elements).
2.  **Codebook B:** 256 entries, each 8-dimensional (stores detail level 1 vectors). Size: $256 \times 8$ bytes.
3.  **Index Texture:** RGB texture of size $(D_x'/4) \times (D_y'/4) \times (D_z'/4)$. Each voxel stores an RGB triplet where R, G, B are 8-bit indices into Codebook A, Codebook B, and a scalar mean value (or color) respectively.
4.  **Header:**
    *   Flag (8-bit): Endianness.
    *   Integer Size (bytes): Machine-independent integer size.
    *   Mode (variable): 1 if data is difference-encoded, 0 if original.
    *   Dimensions ($sizeX, sizeY, sizeZ$): $D_x'/4 \times D_y'/4 \times D_z'/4$.
    *   $scaleDiff0$: Largest absolute value in difference level 0.
    *   $scaleDetl0$: Largest absolute value in detail level 0.
    *   Codebook A data.
    *   Codebook B data.
    *   RGB Triplets data.

**Parameters:**
*   Block Size: $4^3 = 64$ voxels [S2, unofficial].
*   Codebook Size: 256 entries (8-bit indices) [S2, unofficial].
*   Data Type: 8-bit integers for indices and difference values [S2, unofficial].
*   Interpolation: Nearest neighbor only during rendering [S3, unofficial].

**Steps:**

1.  **Preprocessing (Resampling):**
    If original dimensions $D_x, D_y, D_z$ are not powers of two, re-sample the volume to the nearest power-of-two dimensions (e.g., $60 \times 120 \times 79 \to 64 \times 128 \times 128$) using trilinear interpolation [S2, unofficial]. Let the re-sampled data be $S_t'$.

2.  **Preprocessing (Time Difference):**
    For time-varying data, exploit temporal redundancy by computing the difference between consecutive time steps:
    $$ D_t = S_{t+1}' - S_t' $$
    This results in sparse data with many zeros, improving VQ performance [S2, unofficial].

3.  **Hierarchical Decomposition:**
    Divide the difference volume (or static volume) into disjoint $4^3$ blocks. For each block, perform a multiresolution decomposition:
    *   **Step 3.1 (Level 0):** Divide the $4^3$ block into 8 disjoint $2^3$ sub-blocks. Compute the mean value of each $2^3$ sub-block. These 8 mean values form "Detail Level 0" (a down-sampled $2^3$ block).
    *   **Step 3.2 (Level 1):** Compute the differences between the original 64 samples in the $4^3$ block and their corresponding mean values from Step 3.1. Store these 64 differences in an array. The mapping index for location $(x, y, z)$ in the $4^3$ block is:
        $$ idx = x + 4 \cdot y + 16 \cdot z $$
    *   **Step 3.3 (Mean):** Apply the same mean-computation process to the Detail Level 0 block ($2^3$). This yields one overall mean value for the entire $4^3$ block and 8 difference values (samples in Detail Level 0 relative to its mean).
    *   **Step 3.4 (Storage Layout):** Replace the values in Detail Level 0 with the 8 differences from Step 3.3 to save memory. The block is represented in memory as:
        *   Difference Level 1: Array of size 64.
        *   Detail Level 0: Array of size 8.
        *   Detail Level 1: Single variable (the overall mean).

4.  **Covariance Analysis & Codebook Generation:**
    *   Perform covariance analysis on the vectors from the decomposition steps to determine initial codebook entries [S3, unofficial].
    *   **Codebook A (64-dim):** Generated from the 64-dimensional difference vectors (Step 3.2).
    *   **Codebook B (8-dim):** Generated from the 8-dimensional detail vectors (Step 3.3).
    *   Refine codebooks using the LBG algorithm [S2, S6, unofficial]. The number of entries is fixed to 256 [S2, unofficial].

5.  **Quantization:**
    *   For each decomposed block, find the best matching codeword in Codebook A for the 64-dim vector and Codebook B for the 8-dim vector.
    *   Store the 8-bit indices for Codebook A and Codebook B.
    *   Store the overall mean value (or RGB color triplet if applicable) in the index texture [S2, unofficial].

6.  **GPU Rendering (Decoding):**
    *   Upload the index texture and Codebooks A and B to GPU memory as 2D dependent textures [S3, unofficial].
    *   For each fragment, use the Pixel Shader API (DirectX9/OpenGL ARB fragment program) to look up the codewords using the RGB indices.
    *   Reconstruct the scalar value by combining the mean, the decoded detail vectors, and the difference vectors according to the hierarchical structure [S3, unofficial].
    *   Use early z-test to discard fragments decoding empty space [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Encoder
def decompose_block(block_64):
    # block_64 is a list of 64 floats
    # Step 3.1: Create 8 sub-blocks of 8 elements each
    sub_blocks = []
    for i in range(8):
        sub_blocks.append(block_64[i*8 : (i+1)*8])
    
    # Compute means of sub-blocks -> Detail Level 0 (8 values)
    detail_level_0 = [sum(sb)/8.0 for sb in sub_blocks]
    
    # Step 3.2: Differences between original and corresponding means
    # Note: The mean is constant for the 8 elements in the sub-block
    diff_level_1 = []
    for i, sb in enumerate(sub_blocks):
        mean_val = detail_level_0[i]
        for val in sb:
            diff_level_1.append(val - mean_val)
            
    # Step 3.3: Mean of Detail Level 0 -> Overall Mean
    overall_mean = sum(detail_level_0) / 8.0
    
    # Step 3.4: Differences of Detail Level 0 relative to Overall Mean
    detail_level_1_diffs = [val - overall_mean for val in detail_level_0]
    
    return diff_level_1, detail_level_1_diffs, overall_mean

def generate_codebooks(vectors_64, vectors_8):
    # vectors_64: list of 64-dim vectors from diff_level_1
    # vectors_8: list of 8-dim vectors from detail_level_1_diffs
    # Perform covariance analysis and LBG refinement
    # Return codebook_A (256 x 64) and codebook_B (256 x 8)
    pass

def quantize_block(diff_64, diff_8, mean_val, codebook_A, codebook_B):
    # Find index in codebook_A closest to diff_64
    idx_a = argmin([dist(diff_64, codeword) for codeword in codebook_A])
    # Find index in codebook_B closest to diff_8
    idx_b = argmin([dist(diff_8, codeword) for codeword in codebook_B])
    return idx_a, idx_b, mean_val

# Decoder (Shader-like logic)
def decode_fragment(idx_r, idx_g, idx_b, codebook_A, codebook_B, mean_val):
    # R -> idx_a, G -> idx_b, B -> mean (or color)
    vec_a = codebook_A[idx_r]
    vec_b = codebook_B[idx_g]
    
    # Reconstruct the 64 differences
    # Note: The exact reconstruction requires reversing the hierarchical steps
    # This is a simplified representation of the lookup
    reconstructed_diffs = vec_a
    
    # The final value is derived from mean and reconstructed diffs
    # In practice, this is done per-fragment in the shader
    return mean_val # Simplified return
```

## Subtopics

### [Temporal/Spatial Vector Quantization](5743-temporal-spatial-vector-quantization.md)
Temporal/Spatial Vector Quantization (TSVQ) is a compression technique for multivariate time-varying volumetric data that groups voxels into blocks either along the temporal axis or the spatial axis to apply vector quantization algorithms. It differs from static volume compression by explicitly exploiting temporal coherence, and from purely spatial methods by allowing the grouping dimension to be chosen based on the data's coherence characteristics.

### Hierarchical Vector Quantization on Delta
This variant first performs delta encoding of adjacent time steps, then learns the resulting difference data with hierarchical vector quantization [S5]. This approach leverages the sparsity of temporal differences to improve compression efficiency.

## Operating characteristics
**Time Complexity:** The computational complexity of HVQ is $O(N \times M)$, where $N$ is the number of blocks and $M$ is the number of dictionaries (hierarchy levels) [S4, S5]. For $M=1$, it reduces to standard VQ complexity. Pre-processing time is critical, especially for massive volumes, as dictionary generation is costly [S4].

**Memory Complexity:** The compressed size is significantly smaller than the original. For example, a $256^3$ dataset (16 MB) is encoded into 0.78 MB [S3, unofficial]. The codebooks occupy fixed memory (16384 bytes for A, 2048 bytes for B) [S2, unofficial].

**Performance:**
*   **Compression Ratio:** Achieves ~20:1 to 25:1 compression ratios with minimal artifacts [S2, S3, unofficial]. Visible Human data achieved 25.29:1 with 26.02 dB SNR [S3, unofficial].
*   **Rendering Speed:** On an ATI 9700, rendering a 256^3 uncompressed dataset (16 MB) runs at 11 fps, while a 70 MB compressed sequence of a 1.4 GB shock wave simulation runs at 24 fps at a 512^2 viewport [S3, unofficial].
*   **Quality:** Bits-per-sample (bps) of 0.63 (Chameleon), 0.17 (Supernova), and 0.71 (Turbulence) [S1, unofficial]. Debloked SSIM scores: 0.78 (Chameleon), 0.76 (Supernova), 0.89 (Turbulence) [S1, unofficial].

**Strengths:**
*   Minimal CPU-GPU data transfer due to on-the-fly GPU decoding [S2, unofficial].
*   Handles large datasets that exceed texture memory [S3, unofficial].
*   Exploits temporal coherence for time-varying data [S3, unofficial].
*   Early z-test optimization skips empty space [S3, unofficial].

**Limitations:**
*   Restricted to nearest-neighbor interpolation; trilinear interpolation is not natively supported and requires decoding 8 neighbors, which slows performance [S3, unofficial].
*   Dictionary size imposes a hard limit on quality and compression rate [S4].
*   Pre-processing time is high [S4].
*   Codebooks handle only 64 time instances in some implementations; switching codebooks can cause rendering stalls [S6, unofficial].
*   Highly sparse data allows lossless compression, but most volumetric data is not sparse enough for this [S2, unofficial].

## Background
HVQ was first introduced in volume rendering by [NH92, NH93] [S4]. The first GPU implementation was proposed by Schneider et al. in 2003 [S5, S4]. Fout et al. later combined HVQ with domain transformation [S5]. It is used in real-time rendering of large time-varying volumetric datasets, such as shock wave simulations and medical imaging, where minimizing data transfer and maximizing frame rates are critical [S2, S3, unofficial].

## Sources
[S1] Real-time Deblocked GPU Rendering of Compressed Volumes, https://doi.org/10.2312/vmv.20141290, fetched 2026-10-26 (unofficial)
[S2] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S3] Compression Domain Volume Rendering, https://www.cs.cit.tum.de/fileadmin/w00cfj/cg/Research/Publications/2003/Compression_Domain_Volume_Rendering/vis03-vq.pdf, fetched 2026-10-26 (unofficial)
[S4] PDF State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S5] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S6] PDF Volume Rendering of Time-Varying Scalar Fields on Unstructured Meshes, https://www.sci.utah.edu/~stevec/papers/tvrc.pdf, fetched 2026-10-27 (unofficial)
