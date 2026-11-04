# Karhunen-Loeve Transform (KLT)
Also called: Eigen-vector Transform, PCA, PCA-like decorrelation

## Overview
The Karhunen-Loeve Transform (KLT) is a linear decorrelation technique used in volume data compression to remove redundancy by estimating the covariance matrix and computing its eigenvectors [S1][S2, unofficial]. It is closely related to Principal Component Analysis (PCA) and provides superior decorrelation quality compared to the Discrete Cosine Transform (DCT) [S1][S2, unofficial]. Unlike orthogonal basis methods like DCT, KLT bases are data-dependent and not mutually independent, which prevents the use of fast forward and inverse transforms [S1][S2, unofficial]. To mitigate the computational cost of these transforms, KLT is often combined with Vector Quantization (VQ) and block-wise decomposition, shifting the heavy reconstruction work to an offline preprocessing stage [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A volume dataset $V$, typically subdivided into small blocks $B$.
- Output: Compressed data consisting of VQ indices (and potentially residual indices) and precomputed partial reprojection data (in VQ variants).

**Data Structures:**
- **Covariance Matrix ($C$):** A square matrix where $C_{ij} = E[(x_i - \mu_i)(x_j - \mu_j)]$ for the data block.
- **Eigenvectors ($E$):** The columns of the transformation matrix, derived from the eigen-decomposition of $C$.
- **Codebooks ($D_k$):** A set of vectors (codewords) used for Vector Quantization. In multi-band variants, there are separate codebooks for low, band, and high-frequency components.

**Steps:**
1. **Block Decomposition:** Subdivide the volume dataset into small, non-overlapping blocks [S1][S2, unofficial]. This overcomes the issue of dependent bases across large datasets [S1][S2, unofficial].
2. **Covariance Estimation:** For each block (or the entire dataset if memory permits), compute the mean vector $\mu$ and the covariance matrix $C$ [S1][S2, unofficial].
3. **Eigen-decomposition:** Compute the eigenvectors of $C$. These eigenvectors form the KLT basis [S1][S2, unofficial].
4. **Transform:** Apply the KLT to the data blocks. Let $x$ be a data vector in a block. The transformed vector $y$ is:
   $$ y = E^T (x - \mu) $$
   where $E$ is the matrix of eigenvectors [S1][S2, unofficial].
5. **Vector Quantization (VQ):** Encode the transformed coefficients using a VQ codebook. Find the codeword $c \in D$ that minimizes the distance to $y$ [S1][S2, unofficial].
6. **Residual Encoding (Multi-band variant):** If the quantization error (residual) exceeds a threshold, encode the residual using a secondary dictionary [S1]. Repeat this process to identify three bands (low-, band-, high-pass) encoded with three separate dictionaries [S1].
7. **Precomputation (VQ variant):** To avoid the computationally intensive inverse KLT during rendering, precompute partial reprojections and store them in the codebooks instead of the coefficients [S1]. This allows reconstruction to be performed via lookup rather than matrix multiplication [S1].

**Parameters:**
- **Block Size:** Not given in the sources. Small blocks are used to handle dependent bases [S1][S2, unofficial].
- **Threshold:** Determines when a residual is encoded with a second dictionary. Not given in the sources.
- **Number of Bands:** Typically three (low, band, high) in the multi-band variant [S1].

**Edge Cases and Pitfalls:**
- **Computational Intensity:** The inverse KLT is computationally intensive [S1]. This is the primary motivation for using VQ and precomputed reprojections [S1].
- **Dependent Bases:** The bases are not independent, preventing fast transforms [S1][S2, unofficial]. Block-wise decomposition is required to manage this [S1][S2, unofficial].
- **Precision:** Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def klt_compress(volume, block_size, codebooks):
    """
    Compress volume using KLT and VQ.
    volume: 3D array of voxel values.
    block_size: Size of each block (e.g., 4x4x4).
    codebooks: List of codebooks [D_low, D_band, D_high].
    """
    compressed_data = []
    # Step 1: Subdivide volume into blocks
    blocks = extract_blocks(volume, block_size)
    
    for block in blocks:
        # Step 2 & 3: Compute covariance and eigenvectors for this block
        # Note: In practice, eigenvectors are often computed once for the dataset
        # or a representative subset. Here we assume per-block for clarity,
        # though [S1] implies a global or block-wise basis.
        mu = mean(block)
        C = compute_covariance(block, mu)
        E = compute_eigenvectors(C)
        
        # Step 4: Transform
        x_flat = block.flatten()
        y = E.T @ (x_flat - mu)
        
        # Step 5: VQ Encoding
        best_idx = -1
        best_dist = infinity
        for idx, codeword in enumerate(codebooks[0]): # Low band
            dist = sum((y - codeword)**2)
            if dist < best_dist:
                best_dist = dist
                best_idx = idx
        
        # Step 6: Residual Encoding (Multi-band)
        residual = y - codebooks[0][best_idx]
        if best_dist > threshold:
            # Encode residual with second dictionary
            best_idx_band = -1
            best_dist_band = infinity
            for idx, codeword in enumerate(codebooks[1]): # Band
                dist = sum((residual - codeword)**2)
                if dist < best_dist_band:
                    best_dist_band = dist
                    best_idx_band = idx
            residual = residual - codebooks[1][best_idx_band]
            if best_dist_band > threshold:
                # Encode residual with third dictionary
                best_idx_high = -1
                best_dist_high = infinity
                for idx, codeword in enumerate(codebooks[2]): # High
                    dist = sum((residual - codeword)**2)
                    if dist < best_dist_high:
                        best_dist_high = dist
                        best_idx_high = idx
                compressed_data.append((best_idx, best_idx_band, best_idx_high))
        else:
            compressed_data.append((best_idx, -1, -1))
            
    return compressed_data

def klt_decompress(compressed_data, codebooks, precomputed_reprojections):
    """
    Decompress using precomputed reprojections to avoid inverse KLT.
    """
    # Reconstruction is mostly offline, prior to rendering [S1][S2, unofficial]
    # The codebooks store partial reprojections instead of coefficients [S1]
    # This implies the 'codewords' in the codebooks are already transformed
    # or pre-projected for rendering.
    
    reconstructed_volume = []
    for indices in compressed_data:
        idx_low, idx_band, idx_high = indices
        # Retrieve precomputed reprojection data from codebooks
        # This step avoids the inverse KLT matrix multiplication [S1]
        data_low = precomputed_reprojections[0][idx_low]
        data_band = precomputed_reprojections[1][idx_band] if idx_band != -1 else 0
        data_high = precomputed_reprojections[2][idx_high] if idx_high != -1 else 0
        
        reconstructed_volume.append(data_low + data_band + data_high)
        
    return reconstructed_volume
```

## Subtopics

### [KLT Decompression with VQ](5374-klt-decompression-with-vq.md)
KLT Decompression with VQ is a hybrid compression method for volume rendering that addresses the high memory bandwidth and computational cost of decoding decorrelated data. It sets itself apart from standard KLT decoding by precomputing the partial reprojections required for reconstruction and encoding them using Vector Quantization (VQ), thereby reducing the runtime decoding complexity.

### Karhunen-Loeve Transform (KLT) with Multi-Band VQ
A method by Fout and Ma [FM07] that subdivides volume into blocks, applies KLT, and encodes low, band, and high-pass components with separate dictionaries [S1]. It uses a threshold to determine if a residual should be encoded with a second or third dictionary, allowing for efficient storage of low-frequency data while retaining detail for high-frequency signals [S1]. The inverse KLT is avoided by precomputing partial reprojections stored in the codebooks [S1].

## Operating characteristics
- **Time Complexity:** The forward KLT and inverse KLT are computationally intensive due to the lack of fast transforms [S1][S2, unofficial]. However, when combined with VQ and precomputed reprojections, the runtime decoding complexity is reduced [S1][S2, unofficial]. The preprocessing step (computing eigenvectors and codebooks) is performed offline [S1][S2, unofficial].
- **Memory Complexity:** Depends on the size of the codebooks and the number of bands. Multi-band variants require storing multiple codebooks [S1].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Block-wise decomposition allows for parallel processing of blocks [S1].
- **Strengths:** Better decorrelation than DCT [S1][S2, unofficial]. Effective for volume data compression when combined with VQ [S1][S2, unofficial].
- **Limitations:** Bases are not independent, preventing fast transforms [S1][S2, unofficial]. Computationally intensive inverse transform [S1].
- **When to use:** When high-quality decorrelation is required and offline preprocessing is acceptable [S1][S2, unofficial].
- **When not to use:** When fast, real-time decoding without precomputed data is required [S1].

## Background
The Karhunen-Loeve Transform is a linear transform that removes redundancy by decorrelating data, closely related to PCA [S1][S2, unofficial]. It was selected by Fout et al. [FM07] for better quality decorrelation in volume rendering [S1]. The method is part of the broader field of Tensor Approximation (TA), which uses SVD/PCA-like tools to approximate volume datasets [S1][S2, unofficial].

## Sources
[S1] State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27 (unofficial)
