# Vector Quantization

Also called: LBG algorithm modification, 基于字典的压缩

## Overview
Vector Quantization (VQ) is a lossy compression technique that maps $n$-dimensional input vectors to a single fixed-length index referencing an entry in a learned or predefined codebook (dictionary) [S1, S3, unofficial][S5]. It solves the problem of high storage requirements for volumetric data by exploiting spatial redundancy, where similar data blocks are represented by a single shared codeword [S5]. Unlike [Quantization](5935-quantization.md), which typically quantizes individual scalar values, VQ quantizes multi-dimensional blocks, achieving higher compression ratios for data with strong local correlations [S1, unofficial][S5]. It sets itself apart from [Triadic frequency band decomposition](828-triadic-frequency-band-decomposition.md) and [Wavelet Kernel Selection](7035-wavelet-kernel-selection.md) by operating directly on spatial blocks rather than frequency coefficients, and from [Fortune's Palette Compression](4988-fortune-s-palette-compression.md) by supporting arbitrary vector dimensions and learned dictionaries rather than just unique scalar values [S5].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ input vectors $\mathbf{X} = \{\mathbf{x}_1, \mathbf{x}_2, \dots, \mathbf{x}_N\}$, where each $\mathbf{x}_i \in \mathbb{R}^d$. In volume rendering, these vectors typically represent small 3D blocks of voxel values (e.g., $4 \times 4 \times 4$) [S1, unofficial][S5].
*   **Codebook:** A set of $K$ codewords $\mathbf{C} = \{\mathbf{c}_1, \mathbf{c}_2, \dots, \mathbf{c}_K\}$, where $K = 2^r$ and $r$ is the number of bits per index [S1, unofficial].
*   **Output:** A compressed representation consisting of the codebook $\mathbf{C}$ and a mapping index array $\mathbf{J} = \{j_1, j_2, \dots, j_N\}$, where $j_i \in \{1, \dots, K\}$ is the index of the best-matching codeword for $\mathbf{x}_i$ [S1, unofficial][S2, S5].

**Data Structures:**
*   **Codebook:** Stored as a list of $K$ vectors of dimension $d$. In GPU implementations, this may be stored in a 3D texture or buffer [S1, unofficial][S2].
*   **Index Array:** A flat array of $r$-bit integers (e.g., 8-bit) storing the index for each input vector [S1, unofficial][S5]. For sparse data, this may be stored as a 3D texture on the GPU [S1, unofficial].

**Algorithm Steps:**

1.  **Block Partitioning:** Divide the input volume into non-overlapping $d$-dimensional blocks. Common block sizes include $4 \times 4 \times 4$ [S6, unofficial]. Let $N$ be the total number of blocks.
2.  **Codebook Initialization:** Initialize the codebook $\mathbf{C}$ with $K$ entries. This can be done using random sampling from the input data or a structured splitting strategy (e.g., Principal Component Analysis splitting) to avoid empty cells [S1, S3, unofficial].
3.  **Codebook Optimization (Training):** Iteratively refine $\mathbf{C}$ to minimize the average distortion $D_{avg}$ [S1, unofficial]:
    $$D_{avg} = \frac{1}{N} \sum_{i=1}^N \|\mathbf{x}_i - \mathbf{c}_{j_i}\|^2_2$$
    where $\|\cdot\|_2$ is the Euclidean norm (squared-distance metric) [S1, S3, unofficial].
    *   **Assignment Step:** For each input vector $\mathbf{x}_i$, find the index $j_i$ of the closest codeword:
        $$j_i = \arg\min_{k \in \{1,\dots,K\}} \|\mathbf{x}_i - \mathbf{c}_k\|^2_2$$
    *   **Update Step:** Recompute each codeword $\mathbf{c}_k$ as the centroid of all input vectors assigned to it:
        $$\mathbf{c}_k = \frac{\sum_{i=1}^N \mathbb{I}(j_i = k) \mathbf{x}_i}{\sum_{i=1}^N \mathbb{I}(j_i = k)}$$
        where $\mathbb{I}(\cdot)$ is the indicator function. If a cell is empty (no vectors assigned), it must be handled (e.g., by reinitializing with a random input vector or using PCA splitting) to avoid the "empty cell problem" [S1, S3, unofficial].
    *   Repeat Assignment and Update steps until convergence or a maximum number of iterations is reached. The Linde-Buzo-Gray (LBG) algorithm is the standard iterative method for this [S1, S3, unofficial].
4.  **Encoding:** For each input vector $\mathbf{x}_i$, compute the index $j_i$ of the best-matching codeword in the final codebook $\mathbf{C}$ [S1, unofficial][S5]. Store $j_i$ in the index array.
5.  **Decoding:** To reconstruct the volume, replace each index $j_i$ in the index array with the corresponding codeword $\mathbf{c}_{j_i}$ from the codebook [S1, unofficial][S5].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vector_quantization_encode(input_vectors, codebook_size, max_iterations):
    """
    Encodes input vectors using Vector Quantization.
    
    Args:
        input_vectors: List of N vectors, each of dimension d.
        codebook_size: Number of codewords K (must be power of 2).
        max_iterations: Maximum number of LBG iterations.
        
    Returns:
        codebook: List of K codewords.
        indices: List of N indices into the codebook.
    """
    N = len(input_vectors)
    d = len(input_vectors[0])
    
    # Step 1: Initialize Codebook
    # Simple initialization: pick K random vectors from input
    # [G] In practice, PCA splitting or k-means++ is preferred to avoid empty cells.
    indices_init = random.sample(range(N), codebook_size)
    codebook = [input_vectors[i] for i in indices_init]
    
    # Step 2: Iterative Optimization (LBG)
    for iteration in range(max_iterations):
        # Assignment Step
        new_indices = []
        for x in input_vectors:
            min_dist = infinity
            best_k = 0
            for k in range(codebook_size):
                # Squared Euclidean distance
                dist = sum((x[j] - codebook[k][j])**2 for j in range(d))
                if dist < min_dist:
                    min_dist = dist
                    best_k = k
            new_indices.append(best_k)
        
        # Update Step
        new_codebook = [None] * codebook_size
        counts = [0] * codebook_size
        
        for i, x in enumerate(input_vectors):
            k = new_indices[i]
            counts[k] += 1
            if new_codebook[k] is None:
                new_codebook[k] = [0.0] * d
            for j in range(d):
                new_codebook[k][j] += x[j]
        
        # Handle empty cells to avoid division by zero
        for k in range(codebook_size):
            if counts[k] == 0:
                # [G] Re-initialize empty codeword with a random input vector
                new_codebook[k] = input_vectors[random.randint(0, N-1)]
            else:
                for j in range(d):
                    new_codebook[k][j] /= counts[k]
        
        codebook = new_codebook
        
        # Check convergence (e.g., change in codebook is small)
        # [G] Not given in sources, but standard practice.
        
    # Step 3: Final Encoding
    final_indices = []
    for x in input_vectors:
        min_dist = infinity
        best_k = 0
        for k in range(codebook_size):
            dist = sum((x[j] - codebook[k][j])**2 for j in range(d))
            if dist < min_dist:
                min_dist = dist
                best_k = k
        final_indices.append(best_k)
        
    return codebook, final_indices

def vector_quantization_decode(indices, codebook):
    """
    Decodes indices back to approximate vectors.
    
    Args:
        indices: List of N indices into the codebook.
        codebook: List of K codewords.
        
    Returns:
        reconstructed_vectors: List of N reconstructed vectors.
    """
    return [codebook[idx] for idx in indices]
```

**Parameters:**
*   **Codebook Size ($K$):** Determines the compression ratio and quality. Larger $K$ yields lower distortion but higher storage for the codebook and indices [S1, unofficial][S5]. A typical bit-size for the index is 8 bits ($K=256$) [S1, unofficial].
*   **Block Size:** The dimensionality of the input vectors. Common sizes are $4 \times 4 \times 4$ [S6, unofficial]. Larger blocks capture more spatial correlation but increase computational cost and codebook size requirements [S5].
*   **Max Iterations:** Controls the training time and convergence of the codebook [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Empty Cell Problem:** The LBG algorithm can result in codewords with no assigned input vectors, leading to wasted codebook entries and increased distortion [S1, S3, unofficial]. This is mitigated by using PCA-based splitting or reinitializing empty cells [S1, S3, unofficial].
*   **Accumulated Error:** In time-varying data, decoding difference frames (P-frames) can accumulate error over time. This is mitigated by periodically sending full frames (I-frames) [S1, unofficial].
*   **Pre-processing Time:** Learning dictionaries for massive volumes requires significant pre-processing time [S5].
*   **GPU Implementation Limitations:** When implementing VQ decoding on GPUs using languages like Cg, shift and bitwise operations are not supported. Integer coordinates must be mapped to float texture coordinates using specific transformations (e.g., $x' = (x * 0.5 + 0.5) * (D_x - 1) / ((D_x - 4) / (D_y - 1)) - (2 * D_x - 8) / 3$) and texture filtering (GL_NEAREST, GL_REPEAT, GL_CLAMP) must be used to simulate integer arithmetic [S1, unofficial].

## Subtopics

### [Hierarchical Vector Quantization](827-hierarchical-vector-quantization.md)
Uses multiple resolution levels (codebooks) to capture coarse and fine details, enabling direct GPU rendering and handling large-scale data.

### [LBG Algorithm](840-lbg-algorithm.md)
The standard iterative algorithm for generating optimal codebooks by minimizing distortion.

### [Progressive Encoding of Time-Varying Sequences](959-progressive-encoding-of-time-varying-sequences.md)
Extends VQ to time-varying data by reusing codebooks and encoding differences (P-frames) to reduce memory and encoding time.

### [Vector Quantization for Volumetric Radiance Fields](1870-vector-quantization-for-volumetric-radiance-fields.md)
Applies VQ to compress color features in NeRF-like grids, allowing per-voxel random access and trade-offs between compression and quality.

### [Fortune's Palette Compression](4988-fortune-s-palette-compression.md)
A lossless variant for sparse data with few unique values, mapping values to a compact palette.

### [Compact3D / C3DGS / EAGLES](6139-compact3d-c3dgs-eagles.md)
Applies VQ to 3D Gaussian Splatting parameters (scale, SH coefficients) for significant size reduction.

### [Codebook Sharing](7563-codebook-sharing.md)
Reuses a single codebook across tree levels or similar images to reduce memory usage.

### [Brood-based Vector Quantization](7595-brood-based-vector-quantization.md)
Applies VQ to residuals in hierarchical tree structures to correct coarse-level errors.

## Operating characteristics
*   **Time Complexity:** Encoding is dominated by the codebook training (LBG), which is $O(N \cdot K \cdot d \cdot I)$, where $I$ is the number of iterations. Decoding is $O(N)$, as it involves a simple index lookup [S1, S3, unofficial].
*   **Memory Complexity:** Storage is $O(K \cdot d + N \cdot r)$, where $r = \log_2 K$ is the bits per index [S1, unofficial][S2].
*   **Performance:** VQ achieves high compression ratios (e.g., ~20:1 for HVQ) with minimal artifacts [S2][S3, unofficial]. Decompression is extremely fast, making it suitable for real-time rendering [S5].
*   **Strengths:** High compression ratio, fast decompression, suitable for real-time GPU rendering [S1, unofficial][S5].
*   **Limitations:** Lossy compression, pre-processing time for codebook generation, limited by dictionary size [S1, unofficial][S5].
*   **When to Use:** Large volumetric data with high spatial redundancy, time-varying data, and scenarios requiring real-time rendering [S1, unofficial][S5].
*   **When Not to Use:** Data with low redundancy or high entropy, where lossless compression is required [S1, unofficial].

## Background
Vector Quantization was first introduced in volume rendering by [NH92, NH93] [S5]. The foundational LBG algorithm was developed by Linde, Buzo, and Gray in 1980, improving upon scalar quantizers by Max (1960) and Lloyd (1982) [S3, unofficial]. It is widely used in volume rendering for its ability to handle large time-varying data with high spatial and temporal redundancy [S1, unofficial].

## Sources
[S1] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S2] Compressing Volumetric Radiance Fields to 1 MB, https://doi.org/10.1109/cvpr52729.2023.00411, fetched 2026-10-26
[S3] Compression Domain Volume Rendering, https://www.cs.cit.tum.de/fileadmin/w00cfj/cg/Research/Publications/2003/Compression_Domain_Volume_Rendering/vis03-vq.pdf, fetched 2026-10-26 (unofficial)
[S4] A 129 FPS Full HD Real-Time Accelerator for 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3683714, fetched 2026-10-27
[S5] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S6] 体数据压缩技术综述, https://www.jcad.cn/article/id/9d3ce9cd-c97c-4fab-9291-a72fbe7ba37e, fetched 2026-10-30 (unofficial)
