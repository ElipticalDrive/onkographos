# Triadic frequency band decomposition

Also called: Hierarchical Data Decomposition, Hierarchical quantization scheme

## Overview
Triadic frequency band decomposition is a lossy compression method for volumetric data that partitions a scalar field into disjoint $4^3$ blocks, decomposes each block into a multiresolution representation of three frequency bands (high-frequency differences, mid-frequency differences, and mean), and encodes these into vectors of length 64, 8, and 1 respectively [S1, unofficial]. It differs from scalar quantization by grouping spatially correlated differences into fixed-length vectors, and from pure transform coding by relying on hardware-friendly texture lookups for decoding [S1, unofficial].

## How it works
**Inputs:** A 3D scalar volume $V$ of dimensions $N_x \times N_y \times N_z$.
**Outputs:** An RGB index texture $I$ of size $(N_x/4) \times (N_y/4) \times (N_z/4)$, and two codebook textures $C_1$ and $C_2$.

**Data Structures:**
1.  **Index Texture $I$:** A 3D texture where each texel corresponds to one $4^3$ block of the original volume.
    *   **R-component:** 8-bit unsigned integer storing the mean value of the block.
    *   **G-component:** 8-bit unsigned integer storing the index into codebook $C_1$ (high-frequency band).
    *   **B-component:** 8-bit unsigned integer storing the index into codebook $C_2$ (mid-frequency band).
2.  **Codebook $C_1$:** A 2D texture of size $256 \times 64$.
    *   Indexed by $s$-coordinate (from $I$.G) and $t$-coordinate (relative address from Address Texture $A$).
    *   Stores 64-component codewords representing high-frequency difference vectors.
3.  **Codebook $C_2$:** A 2D texture of size $256 \times 8$.
    *   Indexed by $s$-coordinate (from $I$.B) and $t$-coordinate (relative address from Address Texture $A$).
    *   Stores 8-component codewords representing mid-frequency difference vectors.
4.  **Address Texture $A$:** A 2D texture of size $4^3$ (matching the encoded texel region size).
    *   **R-component:** Stores 64 different addresses to index $C_1$.
    *   **G-component:** Stores 8 different addresses to index $C_2$.
    *   Mapped via nearest-neighbor interpolation.

**Parameters:**
*   **Block size:** $4^3$ voxels.
*   **Down-sampling factor:** 2, using disjoint sets of $2^3$ voxels for averaging.
*   **Codebook length:** 256 entries for both $C_1$ and $C_2$, allowing indices to be stored as one 8-bit value [S1, unofficial].

**Algorithm Steps:**
1.  **Partition:** Divide the original scalar field $V$ into disjoint blocks of size $4^3$.
2.  **High-Frequency Band (64 components):**
    *   Down-sample the $4^3$ block by a factor of 2 by averaging disjoint sets of $2^3$ voxels each to create a coarser $2^3$ representation.
    *   Calculate the difference between the original 64 data samples and the respective down-sampled value.
    *   Store these differences in a 64-component vector $D_{high}$.
3.  **Mid-Frequency Band (8 components):**
    *   Apply the same down-sampling process to the $2^3$ mean values obtained in step 2 to produce one single value representing the mean of the entire block.
    *   Calculate the difference between the $2^3$ mean values and the final single mean value.
    *   Store these 8 difference values in an 8-component vector $D_{mid}$.
4.  **Mean Band (1 component):**
    *   Store the single mean value of the entire block in a 1-component vector $M$.
5.  **Thresholding:**
    *   Apply thresholding to $D_{high}$ and $D_{mid}$. If detail coefficients are small or zero (indicating homogeneous or empty regions), map the respective vector to the zero vector. This maximizes codeword usage for significant non-zero vectors and simplifies the quantization relaxation process [S1, unofficial].
6.  **Vector Quantization:**
    *   Quantize $D_{high}$ using codebook $C_1$ to find the best matching codeword index $idx_1$.
    *   Quantize $D_{mid}$ using codebook $C_2$ to find the best matching codeword index $idx_2$.
    *   Quantize $M$ directly as an 8-bit value (no codebook lookup required) [S1, unofficial].
7.  **Storage:**
    *   Store $M$ in $I$.R, $idx_1$ in $I$.G, and $idx_2$ in $I$.B for each block.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def triadic_decomposition(volume, codebook_C1, codebook_C2, address_texture_A):
    """
    Decomposes a volume into triadic frequency bands and encodes them.
    Note: Codebook generation (training) is assumed to be pre-computed.
    """
    Nx, Ny, Nz = volume.shape
    # Output index texture dimensions
    I = np.zeros((Nx // 4, Ny // 4, Nz // 4, 3), dtype=np.uint8)
    
    for x in range(0, Nx, 4):
        for y in range(0, Ny, 4):
            for z in range(0, Nz, 4):
                # 1. Extract block
                block = volume[x:x+4, y:y+4, z:z+4]
                
                # 2. High-frequency band (64 components)
                # Down-sample by averaging disjoint 2x2x2 sets
                # Block is 4x4x4. We have 8 such sets.
                # Let's define the 8 centers of the 2x2x2 blocks within the 4x4x4 block
                # Indices for 2x2x2 blocks in a 4x4x4 grid:
                # (0,0,0), (0,0,2), (0,2,0), (0,2,2), (2,0,0), (2,0,2), (2,2,0), (2,2,2)
                coarse_block = np.zeros((2, 2, 2))
                for i in range(2):
                    for j in range(2):
                        for k in range(2):
                            # Average the 2x2x2 region
                            region = block[i*2:(i+1)*2, j*2:(j+1)*2, k*2:(k+1)*2]
                            coarse_block[i, j, k] = np.mean(region)
                
                # Difference vector D_high: original - coarse (broadcasted)
                # coarse needs to be expanded to 4x4x4 to subtract from block
                coarse_expanded = np.repeat(np.repeat(np.repeat(coarse_block, 2, axis=0), 2, axis=1), 2, axis=2)
                D_high = block.astype(float) - coarse_expanded.astype(float)
                D_high = D_high.flatten() # Length 64
                
                # 3. Mid-frequency band (8 components)
                # Down-sample coarse_block (2x2x2) to single mean
                mean_val = np.mean(coarse_block)
                # Difference vector D_mid: coarse - mean (broadcasted)
                coarse_mean_expanded = np.full((2, 2, 2), mean_val)
                D_mid = coarse_block.astype(float) - coarse_mean_expanded.astype(float)
                D_mid = D_mid.flatten() # Length 8
                
                # 4. Mean band
                M = mean_val
                
                # 5. Thresholding
                # Map small/zero vectors to zero vector
                threshold = 1e-6 # Example threshold, implementation dependent
                if np.max(np.abs(D_high)) < threshold:
                    D_high = np.zeros(64)
                if np.max(np.abs(D_mid)) < threshold:
                    D_mid = np.zeros(8)
                    
                # 6. Vector Quantization
                # Find index in C1 for D_high
                idx_1 = find_best_codebook_entry(D_high, codebook_C1)
                # Find index in C2 for D_mid
                idx_2 = find_best_codebook_entry(D_mid, codebook_C2)
                
                # 7. Store in Index Texture
                # M is stored directly as 8-bit. Assuming M is normalized or scaled appropriately.
                # [G] Assuming M is scaled to 0-255 range for storage.
                M_uint8 = np.clip(np.round(M * scale_factor + offset), 0, 255).astype(np.uint8)
                
                I[x//4, y//4, z//4, 0] = M_uint8
                I[x//4, y//4, z//4, 1] = idx_1
                I[x//4, y//4, z//4, 2] = idx_2
                
    return I

def find_best_codebook_entry(vector, codebook):
    """
    Finds the index of the codeword in the codebook that minimizes the distance to the vector.
    codebook is a 2D array of shape (256, len(vector)).
    """
    # [G] Euclidean distance is standard for VQ unless specified otherwise.
    distances = np.linalg.norm(codebook - vector, axis=1)
    return np.argmin(distances)
```

**Edge Cases and Pitfalls:**
*   **Precision:** The mean value is stored as an 8-bit integer. If the original data range is large, quantization error in the mean band may be significant. The sources do not specify the scaling/offset for the mean value storage [S1, unofficial].
*   **Boundaries:** The method assumes the volume dimensions are divisible by 4. If not, boundary handling is not specified in the sources [S1, unofficial].
*   **Degenerate Input:** Homogeneous or empty regions result in zero difference vectors, which are mapped to the zero vector. This is handled by the thresholding step [S1, unofficial].

## Subtopics

### Spectral decomposition and error band partitioning
Spectral decomposition and error band partitioning is a method that computes centered Fourier spectra and partitions frequency coordinates into low, mid, and high bands to measure spectral magnitude reconstruction errors [S2, unofficial]. It differs from the spatial triadic decomposition by operating in the frequency domain using 2D Discrete Fourier Transforms (DFT) rather than spatial averaging [S2, unofficial]. It is chosen when evaluating reconstruction fidelity in the frequency domain rather than compressing spatial data [S2, unofficial].

**Definition:** Computes centered Fourier spectra and partitions frequency coordinates into low, mid, and high bands to measure spectral magnitude reconstruction errors [S2, unofficial].

**Steps:**
1.  Compute the 2D discrete Fourier transform $\mathcal{F}(I)$ of the grayscale image $I$.
2.  Apply `fftshift` to center the zero-frequency component, yielding the centered complex spectrum $\hat{I}$.
3.  Extract the magnitude spectrum $A_I$ from $\hat{I}$.
4.  For each frequency coordinate $(\xi, \eta)$, compute the Euclidean radial distance $D(\xi, \eta)$ from the centered zero-frequency location.
5.  Partition coordinates into three bands: low if $D \leq 30$, mid if $30 < D \leq 80$, high if $D > 80$ [S2, unofficial].

**Formulas:**
$$ \hat{I} = \text{fftshift}(\mathcal{F}(I)), \quad A_I = |\hat{I}| $$
$$ \Omega_{\text{low}} = \{(\xi, \eta) : D(\xi, \eta) \leq 30\}, \quad \Omega_{\text{mid}} = \{(\xi, \eta) : 30 < D(\xi, \eta) \leq 80\}, \quad \Omega_{\text{high}} = \{(\xi, \eta) : D(\xi, \eta) > 80\} $$
[S2, unofficial]

## Operating characteristics
*   **Time Complexity:** Encoding involves partitioning, averaging, and vector quantization. Vector quantization requires comparing each difference vector against all 256 codewords, which is $O(N \cdot L)$ where $N$ is the number of blocks and $L$ is the codebook size (256). Decoding is efficient, relying on texture lookups [S1, unofficial].
*   **Memory Complexity:** The compressed representation is significantly smaller than the original volume. The index texture is $1/4$ the size of the original volume in each dimension (total $1/64$ the voxels), with 3 bytes per texel. Codebooks are small fixed-size textures ($256 \times 64$ and $256 \times 8$) [S1, unofficial].
*   **Measured Performance:** The method was implemented on an ATI 9700 GPU, exploiting per-fragment arithmetic and dependent texture fetches [S1, unofficial].
*   **Strengths:** Maximizes codeword usage for significant vectors via thresholding; enhances fidelity by assembling signals from separate bands; hardware-friendly decoding via texture lookups [S1, unofficial].
*   **Limitations:** Lossy compression; requires pre-computed codebooks; block artifacts may occur at block boundaries; mean value quantization error [S1, unofficial].
*   **When to use:** Suitable for compression domain volume rendering on hardware with efficient texture lookup capabilities [S1, unofficial].
*   **When not to use:** When lossless compression is required or when the volume dimensions are not divisible by 4 without padding [S1, unofficial].

## Background
Triadic frequency band decomposition, also known as hierarchical quantization scheme, was introduced in the paper "Compression Domain Volume Rendering" [S1, unofficial]. It was developed to enable efficient volume rendering by compressing data in a way that is compatible with graphics hardware, specifically leveraging the ATI 9700 GPU's capabilities for per-fragment operations and dependent texture fetches [S1, unofficial]. The method partitions data into triadic frequency bands to exploit spatial correlation and allows for efficient decoding through texture-based codebook lookups [S1, unofficial].

## Sources
[S1] Compression Domain Volume Rendering, https://www.cs.cit.tum.de/fileadmin/w00cfj/cg/Research/Publications/2003/Compression_Domain_Volume_Rendering/vis03-vq.pdf, fetched 2026-10-26 (unofficial)
[S2] EffGS: Efficient and High-Fidelity Gaussian Splatting, https://arxiv.org/abs/2609.39553, fetched 2026-10-29 (unofficial)
