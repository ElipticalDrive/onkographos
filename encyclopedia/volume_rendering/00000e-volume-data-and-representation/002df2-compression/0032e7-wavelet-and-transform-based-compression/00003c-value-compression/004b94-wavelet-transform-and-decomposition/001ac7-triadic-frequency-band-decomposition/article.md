# Triadic frequency band decomposition

Also called: Hierarchical Data Decomposition, Hierarchical quantization scheme

## Overview
Triadic frequency band decomposition is a lossy compression method for volumetric scalar fields that partitions the data into $4^3$ blocks and decomposes each block into a multiresolution representation of three frequency bands: high-frequency differences, mid-frequency differences, and the block mean [S1, unofficial]. It differs from its siblings by using a fixed, hardware-friendly block size and a specific hierarchical downsampling scheme that maps directly to dependent texture fetches for on-chip decoding, whereas [Laplacian Pyramid](2250-laplacian-pyramid.md) uses Gaussian pyramids and [Wavelet Kernel Selection](7035-wavelet-kernel-selection.md) focuses on optimizing basis functions for general wavelet transforms.

## How it works
**Inputs:** A 3D scalar volume $V$ of dimensions $N_x \times N_y \times N_z$.
**Outputs:** An RGB index texture $I$, two codebook textures $C_1$ and $C_2$, and an address texture $A$.

**Data Structures:**
1.  **Index Texture $I$:** A 3D texture of size $(N_x/4) \times (N_y/4) \times (N_z/4)$. Each texel stores three 8-bit values:
    *   $R$: The mean value of the corresponding $4^3$ block.
    *   $G$: An index (0–255) into codebook $C_1$ (high-frequency band).
    *   $B$: An index (0–255) into codebook $C_2$ (mid-frequency band).
2.  **Codebook $C_1$:** A 2D texture of size $256 \times 64$. The $s$-coordinate is the index from $I.G$. The $t$-coordinate is a relative address from texture $A$. It stores 64-component codewords representing high-frequency differences.
3.  **Codebook $C_2$:** A 2D texture of size $256 \times 8$. The $s$-coordinate is the index from $I.B$. The $t$-coordinate is a relative address from texture $A$. It stores 8-component codewords representing mid-frequency differences.
4.  **Address Texture $A$:** A 2D texture of size $4^3$ (matching the encoded texel region size). It is mapped via nearest-neighbor interpolation.
    *   $R$-component: Stores 64 different addresses to index $C_1$.
    *   $G$-component: Stores 8 different addresses to index $C_2$.

**Algorithm Steps:**
1.  **Partitioning:** Divide the input volume $V$ into disjoint blocks of size $4^3$ voxels [S1, unofficial].
2.  **High-Frequency Band Extraction:**
    *   For each block, down-sample by a factor of 2 by averaging disjoint sets of $2^3$ voxels. This produces a $2^3$ voxel grid of mean values.
    *   Calculate the difference between the original 64 voxels in the block and the corresponding down-sampled value (the mean of the $2^3$ voxels they belong to).
    *   Store these 64 difference values in a vector $D_{high}$ [S1, unofficial].
3.  **Mid-Frequency Band Extraction:**
    *   Take the $2^3$ mean values obtained in Step 2.
    *   Down-sample this $2^3$ grid by a factor of 2 by averaging disjoint sets of $2^3$ voxels (which is the entire $2^3$ grid) to produce a single scalar value.
    *   Calculate the difference between the original $2^3$ mean values and this single scalar value.
    *   Store these 8 difference values in a vector $D_{mid}$ [S1, unofficial].
4.  **Mean Band Extraction:**
    *   The single scalar value from Step 3 is the mean of the entire $4^3$ block. Store this in a 1-component vector $M$ [S1, unofficial].
5.  **Thresholding:**
    *   Apply thresholding to $D_{high}$ and $D_{mid}$. If a difference vector is small or zero (indicating homogeneous or empty regions), map the entire vector to the zero vector. This maximizes the usage of codewords for significant non-zero vectors and simplifies quantization [S1, unofficial].
6.  **Vector Quantization:**
    *   Quantize $D_{high}$ using a codebook $C_1$ of length 256, resulting in an index $idx_1$.
    *   Quantize $D_{mid}$ using a codebook $C_2$ of length 256, resulting in an index $idx_2$.
    *   Quantize $M$ as an 8-bit value directly [S1, unofficial].
7.  **Storage:**
    *   Store $M$ in $I.R$, $idx_1$ in $I.G$, and $idx_2$ in $I.B$ for the corresponding block location [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def triadic_decomposition(volume, codebook_C1, codebook_C2, address_texture_A):
    """
    Decomposes a 3D volume into triadic frequency bands.
    volume: 3D array of shape (Nx, Ny, Nz)
    codebook_C1: 2D array of shape (256, 64)
    codebook_C2: 2D array of shape (256, 8)
    address_texture_A: 2D array of shape (64, 64) or similar, storing relative addresses
    """
    Nx, Ny, Nz = volume.shape
    # Output textures
    index_texture_I = np.zeros((Nx // 4, Ny // 4, Nz // 4), dtype=np.uint8)
    
    for x in range(0, Nx, 4):
        for y in range(0, Ny, 4):
            for z in range(0, Nz, 4):
                # 1. Extract 4x4x4 block
                block = volume[x:x+4, y:y+4, z:z+4]
                
                # 2. High-frequency band: 64 components
                # Down-sample by averaging 2x2x2 sets
                # The block is 4x4x4. It consists of 8 disjoint 2x2x2 sets.
                # Each 2x2x2 set has 8 voxels.
                # We compute the mean of each 2x2x2 set.
                # Then we compute the difference of each of the 64 voxels 
                # from the mean of its 2x2x2 set.
                
                # Reshape to (8, 8) to represent the 8 sets of 8 voxels
                # Note: The exact mapping of voxels to sets depends on memory layout,
                # but logically it's 8 groups of 8.
                block_flat = block.flatten() # 64 elements
                
                # Calculate means of 2x2x2 blocks
                # Assuming standard row-major flattening, we group every 8 elements
                # as one 2x2x2 block? No, 2x2x2 is 8 voxels.
                # The 64 voxels are partitioned into 8 groups of 8.
                means_2x2x2 = []
                for i in range(0, 64, 8):
                    group = block_flat[i:i+8]
                    means_2x2x2.append(np.mean(group))
                
                # High-freq differences: original - corresponding mean
                D_high = np.zeros(64)
                for i in range(64):
                    group_idx = i // 8
                    D_high[i] = block_flat[i] - means_2x2x2[group_idx]
                
                # Thresholding for D_high
                if np.all(np.abs(D_high) < threshold_high): # threshold_high is implementation detail [G]
                    D_high = np.zeros(64)
                
                # Quantize D_high
                idx_1 = quantize_vector(D_high, codebook_C1)
                
                # 3. Mid-frequency band: 8 components
                # The means_2x2x2 are 8 values.
                # Down-sample these 8 values by averaging all of them to get 1 value.
                mean_block = np.mean(means_2x2x2)
                
                # Mid-freq differences: means_2x2x2 - mean_block
                D_mid = np.array(means_2x2x2) - mean_block
                
                # Thresholding for D_mid
                if np.all(np.abs(D_mid) < threshold_mid): # threshold_mid is implementation detail [G]
                    D_mid = np.zeros(8)
                
                # Quantize D_mid
                idx_2 = quantize_vector(D_mid, codebook_C2)
                
                # 4. Mean band
                M = mean_block
                
                # 5. Store in Index Texture
                # M is stored as 8-bit value. Assuming M is scaled/quantized to 0-255 range.
                # In practice, M might need normalization if it's not already 8-bit compatible.
                index_texture_I[x//4, y//4, z//4, 0] = np.clip(np.round(M), 0, 255).astype(np.uint8)
                index_texture_I[x//4, y//4, z//4, 1] = idx_1
                index_texture_I[x//4, y//4, z//4, 2] = idx_2
                
    return index_texture_I

def quantize_vector(vector, codebook):
    """
    Finds the index of the codeword in the codebook that minimizes the distance to the vector.
    codebook: array of shape (256, vector_length)
    """
    min_dist = float('inf')
    best_idx = 0
    for i in range(256):
        dist = np.linalg.norm(vector - codebook[i])
        if dist < min_dist:
            min_dist = dist
            best_idx = i
    return best_idx
```

**Parameters:**
*   Block size: $4^3$ voxels [S1, unofficial].
*   Down-sampling factor: 2, using disjoint sets of $2^3$ voxels for averaging [S1, unofficial].
*   Codebook length: 256 entries for both $C_1$ and $C_2$, allowing indices to be stored as one 8-bit value [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Homogeneous Regions:** Thresholding maps zero-difference vectors to the zero vector. This is critical for maximizing codeword efficiency [S1, unofficial].
*   **Precision:** Mean values are stored directly as 8-bit values. If the original data has a wider range, scaling or offsetting is required before storage [S1, unofficial].
*   **Boundary Conditions:** The volume dimensions $N_x, N_y, N_z$ must be divisible by 4. If not, padding or partial block handling is required (not specified in sources) [G].

## Subtopics

### Spectral decomposition and error band partitioning
This variant computes centered Fourier spectra and partitions frequency coordinates into low, mid, and high bands to measure spectral magnitude reconstruction errors [S2, unofficial]. It is used for error analysis rather than direct compression.
*   **Definition:** Computes the 2D discrete Fourier transform (DFT) of an image, centers the spectrum, and partitions frequency coordinates based on radial distance from the zero-frequency component [S2, unofficial].
*   **Steps:**
    1.  Compute the 2D DFT $\mathcal{F}(I)$ of the grayscale image $I$.
    2.  Apply `fftshift` to center the zero-frequency component, yielding the centered complex spectrum $\hat{I}$.
    3.  Extract the magnitude spectrum $A_I = |\hat{I}|$.
    4.  For each frequency coordinate $(\xi, \eta)$, compute the Euclidean radial distance $D(\xi, \eta)$ from the centered zero-frequency location.
    5.  Partition coordinates into three bands:
        *   Low: $D(\xi, \eta) \leq 30$
        *   Mid: $30 < D(\xi, \eta) \leq 80$
        *   High: $D(\xi, \eta) > 80$
*   **Formulas:**
    $$ \hat{I} = \text{fftshift}(\mathcal{F}(I)), \quad A_I = |\hat{I}| $$
    $$ \Omega_{\text{low}} = \{(\xi, \eta) : D(\xi, \eta) \leq 30\}, \quad \Omega_{\text{mid}} = \{(\xi, \eta) : 30 < D(\xi, \eta) \leq 80\}, \quad \Omega_{\text{high}} = \{(\xi, \eta) : D(\xi, \eta) > 80\} $$
*   **When to choose:** Use for evaluating reconstruction fidelity in frequency space, not for real-time compression [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Encoding is $O(N)$ where $N$ is the number of voxels, as each voxel is processed once in a block. Vector quantization is $O(N \cdot K)$ where $K=256$ is the codebook size, which can be optimized with search algorithms.
*   **Memory Complexity:** The compressed representation is $1/4$ the size of the original volume in terms of texels (since each $4^3$ block is represented by one texel in $I$), plus the size of the codebooks ($256 \times 64 + 256 \times 8$ floats/bytes) and the address texture ($4^3$).
*   **Performance:** Designed for hardware acceleration using dependent texture fetches on GPUs (e.g., ATI 9700) [S1, unofficial]. Decoding is efficient as it involves simple texture lookups and additions.
*   **Strengths:** High compression ratio for homogeneous data due to thresholding; hardware-friendly decoding; multi-resolution access via the hierarchical structure.
*   **Limitations:** Fixed block size limits adaptability; vector quantization requires pre-computed codebooks; lossy compression introduces artifacts in non-homogeneous regions.
*   **When to use:** Real-time volume rendering on hardware with limited bandwidth and texture units; scenes with large homogeneous regions.
*   **When not to use:** Scenes with fine, high-frequency details throughout; lossless compression requirements.

## Background
Triadic frequency band decomposition was introduced in the context of compression domain volume rendering, specifically targeting hardware acceleration on early GPU architectures like the ATI 9700 [S1, unofficial]. It was developed to leverage the capabilities of dependent texture fetches and per-fragment arithmetic operations for efficient on-chip decoding [S1, unofficial]. The method is part of a broader family of wavelet and transform-based compression techniques for volumetric data.

## Sources
[S1] Compression Domain Volume Rendering, https://www.cs.cit.tum.de/fileadmin/w00cfj/cg/Research/Publications/2003/Compression_Domain_Volume_Rendering/vis03-vq.pdf, fetched 2026-10-26 (unofficial)
[S2] EffGS: Efficient and High-Fidelity Gaussian Splatting, https://arxiv.org/abs/2609.39553, fetched 2026-10-29 (unofficial)
