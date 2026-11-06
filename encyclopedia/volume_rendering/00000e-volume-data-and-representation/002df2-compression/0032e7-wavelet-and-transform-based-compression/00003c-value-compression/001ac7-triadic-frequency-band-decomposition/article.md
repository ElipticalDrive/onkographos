# Triadic frequency band decomposition

Also called: Hierarchical Data Decomposition, Hierarchical quantization scheme

## Overview
Triadic frequency band decomposition is a lossy compression method for volumetric scalar fields that partitions data into disjoint blocks and decomposes each block into three hierarchical frequency bands (high-frequency details, mid-frequency details, and mean) [S1, unofficial]. It differs from standard scalar quantization by encoding spatially correlated differences into low-dimensional vectors, and from pure transform coding by using discrete codebooks for efficient hardware decoding [S1, unofficial]. The method is designed for compression domain rendering, allowing reconstruction of the volume data directly on the GPU via texture lookups and arithmetic operations [S1, unofficial].

## How it works
**Inputs:** A 3D scalar field of size $N_x \times N_y \times N_z$.
**Outputs:** An RGB index texture $I$, two codebook textures $C_1$ and $C_2$, and an address texture $A$.

**Data Structures and Layout:**
1.  **Index Texture $I$:** A 3D texture of size $(N_x/4) \times (N_y/4) \times (N_z/4)$. Each texel stores three 8-bit values:
    *   **R-component:** The mean value of the corresponding $4^3$ block.
    *   **G-component:** An 8-bit index into codebook $C_1$ (high-frequency band).
    *   **B-component:** An 8-bit index into codebook $C_2$ (mid-frequency band).
2.  **Codebook $C_1$:** A 2D texture of size $256 \times 64$. The s-coordinate is the index from $I$'s G-component. The t-coordinate is determined by the Address Texture $A$. It stores 64-component codewords representing high-frequency differences.
3.  **Codebook $C_2$:** A 2D texture of size $256 \times 8$. The s-coordinate is the index from $I$'s B-component. The t-coordinate is determined by the Address Texture $A$. It stores 8-component codewords representing mid-frequency differences.
4.  **Address Texture $A$:** A 2D texture of size $4^3$ (matching the encoded texel region size).
    *   **R-component:** Stores 64 different addresses to index $C_1$.
    *   **G-component:** Stores 8 different addresses to index $C_2$.
    *   Mapped via nearest-neighbor interpolation.

**Algorithm Steps:**
1.  **Block Partitioning:** Partition the original scalar field into disjoint blocks of size $4^3$ voxels [S1, unofficial].
2.  **First Decomposition (High-Frequency):**
    *   Down-sample the $4^3$ block by a factor of two by averaging disjoint sets of $2^3$ voxels each. This produces a $2^3$ mean representation [S1, unofficial].
    *   Calculate the difference between the original 64 data samples and the respective down-sampled value (replicated to $4^3$ size).
    *   Store these 64 difference values in a 64-component vector $V_{high}$ [S1, unofficial].
3.  **Second Decomposition (Mid-Frequency):**
    *   Apply the same down-sampling process to the $2^3$ mean values from Step 2. This produces one single value representing the mean of the entire block [S1, unofficial].
    *   Calculate the difference between the $2^3$ mean values and the final single mean value (replicated to $2^3$ size).
    *   Store these 8 difference values in an 8-component vector $V_{mid}$ [S1, unofficial].
4.  **Mean Storage:** Store the single mean value in a 1-component vector $V_{mean}$ [S1, unofficial].
5.  **Thresholding:** Apply thresholding to $V_{high}$ and $V_{mid}$. If detail coefficients are small or zero (indicating homogeneous or empty regions), map the respective vector to the zero vector. This maximizes codeword usage for significant non-zero vectors and simplifies quantization [S1, unofficial].
6.  **Vector Quantization:**
    *   Quantize $V_{high}$ using a codebook $C_1$ of length 256. The result is an 8-bit index $idx_1$ [S1, unofficial].
    *   Quantize $V_{mid}$ using a codebook $C_2$ of length 256. The result is an 8-bit index $idx_2$ [S1, unofficial].
    *   The mean $V_{mean}$ is stored directly as an 8-bit value [S1, unofficial].
7.  **Assembly:** Assemble the RGB index texture $I$ where each texel contains $(V_{mean}, idx_1, idx_2)$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def triadic_decomposition(volume, block_size=4):
    """
    Decomposes a 3D volume into triadic frequency bands.
    volume: 3D numpy array of shape (Nx, Ny, Nz)
    Returns: mean_texture, idx1_texture, idx2_texture, address_texture
    """
    Nx, Ny, Nz = volume.shape
    # Ensure dimensions are divisible by block_size
    assert Nx % block_size == 0 and Ny % block_size == 0 and Nz % block_size == 0
    
    # Pre-compute address texture for decoding (simplified for encoding context)
    # In practice, this is generated to map local coordinates to codebook indices
    address_texture = generate_address_texture(block_size)
    
    # Initialize output textures
    # Size is reduced by block_size factor
    out_mean = np.zeros((Nx // block_size, Ny // block_size, Nz // block_size), dtype=np.uint8)
    out_idx1 = np.zeros((Nx // block_size, Ny // block_size, Nz // block_size), dtype=np.uint8)
    out_idx2 = np.zeros((Nx // block_size, Ny // block_size, Nz // block_size), dtype=np.uint8)
    
    # Codebooks (assumed pre-trained or provided)
    # C1: shape (256, 64), C2: shape (256, 8)
    # For implementation, we assume a quantization function exists
    def vector_quantize(vector, codebook):
        # Find index of codeword in codebook that minimizes distance to vector
        # Returns index in [0, 255]
        pass 

    for x in range(0, Nx, block_size):
        for y in range(0, Ny, block_size):
            for z in range(0, Nz, block_size):
                # Extract block
                block = volume[x:x+block_size, y:y+block_size, z:z+block_size]
                
                # Step 1: High-frequency band (64 components)
                # Down-sample by averaging 2x2x2 sets
                # block shape: (4, 4, 4)
                # downsampled_2x2x2 shape: (2, 2, 2)
                downsampled_2 = np.zeros((2, 2, 2))
                for i in range(2):
                    for j in range(2):
                        for k in range(2):
                            # Average the 2x2x2 cube corresponding to this voxel
                            downsampled_2[i, j, k] = np.mean(
                                block[2*i:2*i+2, 2*j:2*j+2, 2*k:2*k+2]
                            )
                
                # Calculate high-frequency differences
                # Expand downsampled_2 to 4x4x4 for subtraction
                expanded_2 = np.repeat(np.repeat(np.repeat(downsampled_2, 2, axis=0), 2, axis=1), 2, axis=2)
                high_freq_vec = (block - expanded_2).flatten() # 64 components
                
                # Thresholding for high freq
                # Map small/zero vectors to zero vector
                if np.max(np.abs(high_freq_vec)) < threshold_high:
                    high_freq_vec = np.zeros_like(high_freq_vec)
                
                # Quantize high freq
                idx1 = vector_quantize(high_freq_vec, C1)
                
                # Step 2: Mid-frequency band (8 components)
                # Down-sample downsampled_2 (2x2x2) to single mean
                mean_val = np.mean(downsampled_2)
                
                # Calculate mid-frequency differences
                # Expand mean_val to 2x2x2
                expanded_mean_2 = np.full((2, 2, 2), mean_val)
                mid_freq_vec = (downsampled_2 - expanded_mean_2).flatten() # 8 components
                
                # Thresholding for mid freq
                if np.max(np.abs(mid_freq_vec)) < threshold_mid:
                    mid_freq_vec = np.zeros_like(mid_freq_vec)
                
                # Quantize mid freq
                idx2 = vector_quantize(mid_freq_vec, C2)
                
                # Store results
                # Mean is stored directly as 8-bit value
                out_mean[x//block_size, y//block_size, z//block_size] = np.clip(mean_val, 0, 255).astype(np.uint8)
                out_idx1[x//block_size, y//block_size, z//block_size] = idx1
                out_idx2[x//block_size, y//block_size, z//block_size] = idx2
                
    return out_mean, out_idx1, out_idx2, address_texture
```

**Parameters:**
*   **Block Size:** $4^3$ voxels [S1, unofficial].
*   **Down-sampling Factor:** 2, using disjoint sets of $2^3$ voxels for averaging [S1, unofficial].
*   **Codebook Length:** 256 entries for both $C_1$ and $C_2$, allowing indices to be stored as one 8-bit value [S1, unofficial].
*   **Thresholds:** Not given in the sources. [G] Typical values depend on the noise floor and desired compression ratio; often set to a small epsilon or derived from the standard deviation of the training data.

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The method assumes the volume dimensions are divisible by the block size ($4^3$). Padding or cropping may be required for arbitrary sizes [S1, unofficial].
*   **Precision:** Mean values are stored as 8-bit values. If the original data has higher precision, quantization error is introduced at the mean level [S1, unofficial].
*   **Degenerate Input:** Homogeneous or empty regions result in zero difference vectors, which map to a specific "zero" codeword. This is handled by the thresholding step [S1, unofficial].

## Subtopics

### Spectral decomposition and error band partitioning
Spectral decomposition and error band partitioning is a variant that computes centered Fourier spectra and partitions frequency coordinates into low, mid, and high bands to measure spectral magnitude reconstruction errors [S2, unofficial]. It differs from the spatial block-based triadic decomposition by operating in the frequency domain using 2D Discrete Fourier Transforms (DFT) [S2, unofficial]. It is chosen when spectral analysis is required for error measurement rather than spatial compression [S2, unofficial].

**Definition:** Computes centered Fourier spectra and partitions frequency coordinates into low, mid, and high bands to measure spectral magnitude reconstruction errors [S2, unofficial].

**Steps:**
1.  Compute the 2D discrete Fourier transform $\mathcal{F}(I)$ of the grayscale image $I$ [S2, unofficial].
2.  Apply `fftshift` to center the zero-frequency component, yielding the centered complex spectrum $\hat{I}$ [S2, unofficial].
3.  Extract the magnitude spectrum $A_I$ from $\hat{I}$ [S2, unofficial].
4.  For each frequency coordinate $(\xi, \eta)$, compute the Euclidean radial distance $D(\xi, \eta)$ from the centered zero-frequency location [S2, unofficial].
5.  Partition coordinates into three bands: low if $D \leq 30$, mid if $30 < D \leq 80$, high if $D > 80$ [S2, unofficial].

**Formulas:**
$$ \hat{I} = \text{fftshift}(\mathcal{F}(I)), \quad A_I = |\hat{I}| $$
$$ \Omega_{\text{low}} = \{(\xi, \eta) : D(\xi, \eta) \leq 30\}, \quad \Omega_{\text{mid}} = \{(\xi, \eta) : 30 < D(\xi, \eta) \leq 80\}, \quad \Omega_{\text{high}} = \{(\xi, \eta) : D(\xi, \eta) > 80\} $$

## Operating characteristics
*   **Time Complexity:** Encoding involves block-wise operations (averaging, subtraction, thresholding, vector quantization). The complexity is linear with respect to the number of voxels, $O(N_x N_y N_z)$, assuming constant-time codebook lookup [S1, unofficial].
*   **Memory Complexity:** The compressed representation size is $3 \times (N_x/4)(N_y/4)(N_z/4)$ bytes for the index texture, plus the size of the codebooks ($256 \times 64 \times \text{bytes} + 256 \times 8 \times \text{bytes}$) and address texture [S1, unofficial]. This results in a significant compression ratio compared to the original $N_x N_y N_z$ voxels.
*   **Measured Performance:** Not given in the sources. [S1, unofficial]
*   **Scaling:** The method scales well with volume size as it is block-based. The GPU rendering aspect allows for efficient parallel reconstruction [S1, unofficial].
*   **Strengths:** High compression ratios due to vector quantization and thresholding; efficient hardware decoding via texture lookups; enhanced fidelity by assembling signal from multiple bands [S1, unofficial].
*   **Limitations:** Lossy compression; requires pre-computed codebooks; block artifacts may be visible if thresholding is too aggressive; limited to scalar fields [S1, unofficial].
*   **When to use:** Suitable for compression domain volume rendering where memory bandwidth is a bottleneck and high compression ratios are needed [S1, unofficial].
*   **When not to use:** When lossless compression is required or when the data does not exhibit the spatial correlation assumed by the block-based decomposition [S1, unofficial].

## Background
Triadic frequency band decomposition, also known as hierarchical quantization, was introduced in the context of compression domain volume rendering [S1, unofficial]. It exploits the capabilities of modern GPUs (specifically ATI 9700 at the time of publication) to perform arithmetic operations and dependent texture fetches on a per-fragment basis [S1, unofficial]. The method was developed to enable efficient rendering of large volumetric datasets by reducing memory footprint and bandwidth requirements [S1, unofficial].

## Sources
[S1] Compression Domain Volume Rendering, https://www.cs.cit.tum.de/fileadmin/w00cfj/cg/Research/Publications/2003/Compression_Domain_Volume_Rendering/vis03-vq.pdf, fetched 2026-10-26 (unofficial)
[S2] EffGS: Efficient and High-Fidelity Gaussian Splatting, https://arxiv.org/abs/2609.39553, fetched 2026-10-29 (unofficial)
