# Biorthogonal Wavelet Compression

Also called: Biorthogonal spline wavelet, Daubechies wavelet, Multilevel Wavelet Decomposition, Wavelet Coefficient Thresholding, Wavelet Thresholding

## Overview
Biorthogonal Wavelet Compression compresses volumetric data by applying a multilevel wavelet transform and discarding the least significant coefficients to achieve variable compression ratios while preserving reconstruction quality [S1]. It partitions the volume into blocks and transforms them independently using biorthogonal filters, such as the 9/7-tap Daubechies wavelet or biorthogonal spline wavelets [S2]. Unlike tensor approximation methods which follow a multiscale behavior, this method exhibits multiresolution behavior, making it advantageous for reconstructing the overall statistical distribution of a dataset at coarser resolutions [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volumetric dataset $V$.
*   **Output:** A compressed bitstream containing quantized/entropy-encoded wavelet coefficients.

**Data Structures:**
The method operates on blocks of the volume. Each block is decomposed into subbands. The coefficients are stored in a structure that allows for thresholding. In multiresolution variants, an octree structure is used where each node stores its wavelet transform coefficients [S2].

**Parameters:**
*   **Wavelet Basis:** The biorthogonal 9/7 wavelet is the standard choice, as used in JPEG 2000 [S1]. Biorthogonal spline wavelets are also used [S2].
*   **Threshold:** Coefficients below a certain magnitude are set to zero. The threshold value determines the compression ratio.

**Steps:**
1.  **Block Partitioning:** Partition the input volume $V$ into independent blocks [S2].
2.  **Transform:** Apply a multilevel wavelet decomposition to each block using the chosen biorthogonal filter bank (e.g., 9/7-tap Daubechies) [S1][S2]. This results in approximation and detail coefficients.
3.  **Thresholding:** Identify the least significant coefficients. Threshold these coefficients to zero to discard information that contributes least to visual quality [S1].
4.  **Encoding:** Apply lossless encoding (e.g., Run-Length Encoding, Huffman coding, or Arithmetic coding) to the remaining non-zero coefficients and their positions to produce the compressed bitstream [S1].
5.  **Reconstruction:** To render, decode the bitstream, inverse threshold (restore zeros), and apply the inverse wavelet transform to reconstruct the volume [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def biorthogonal_wavelet_compress(volume, wavelet_basis='bior9_7', threshold=0.0):
    """
    Compress a 3D volume using biorthogonal wavelet transform and thresholding.
    
    Args:
        volume: 3D numpy array representing the volumetric data.
        wavelet_basis: String specifying the filter (e.g., 'bior9_7').
        threshold: Float value; coefficients with absolute magnitude < threshold are zeroed.
        
    Returns:
        compressed_data: A structure containing the non-zero coefficients and their indices,
                         ready for entropy encoding.
    """
    # Step 1: Partition volume into blocks (implementation detail depends on block size)
    # For simplicity, assume the transform function handles block boundaries or padding.
    
    # Step 2: Apply Multilevel Wavelet Decomposition
    # The transform is separable along each axis.
    # Let V be the volume.
    # Level 1:
    #   Approximation A1 = LowPass(LowPass(LowPass(V)))
    #   Details D1_x, D1_y, D1_z, etc. = HighPass combinations
    # Recursive decomposition on A1...
    coefficients = apply_wavelet_transform(volume, wavelet_basis)
    
    # Step 3: Thresholding
    # Identify least significant coefficients
    significant_coefficients = {}
    for key, value in coefficients.items():
        if abs(value) >= threshold:
            significant_coefficients[key] = value
        else:
            significant_coefficients[key] = 0 # Discard
            
    # Step 4: Encoding (Placeholder for specific entropy coder)
    # In practice, this involves run-length encoding or arithmetic coding
    # of the sparse coefficient array.
    compressed_data = entropy_encode(significant_coefficients)
    
    return compressed_data

def biorthogonal_wavelet_decompress(compressed_data, volume_shape, wavelet_basis='bior9_7'):
    """
    Reconstruct volume from compressed wavelet data.
    """
    # Step 4 (Inverse): Decode
    coefficients = entropy_decode(compressed_data, volume_shape)
    
    # Step 5: Inverse Transform
    reconstructed_volume = apply_inverse_wavelet_transform(coefficients, wavelet_basis)
    
    return reconstructed_volume
```

**Edge Cases and Pitfalls:**
*   **Boundary Effects:** Since the transform is applied to blocks, boundary artifacts may occur at block edges if not handled with appropriate padding or overlap-add techniques [S2].
*   **Precision:** Thresholding introduces loss. The choice of threshold must balance compression ratio and reconstruction quality [S1].
*   **Degenerate Input:** Very small volumes may not support multiple decomposition levels.

## Subtopics

### Wavelet Octree Multiresolution
A multiresolution approach that builds an octree where each node stores its wavelet transform coefficients, computed using biorthogonal filters (e.g., as in [GWGS02]) [S2]. This structure allows for run-time decoding via an FPGA hardware decoder, enabling efficient adaptive resolution rendering [S2].

## Operating characteristics
*   **Compression Ratio:** Achieves high compression ratios while maintaining good reconstruction quality. On a dental enamel dataset, compression ratios of ~13%, ~2%, ~0.4%, and ~0.1% were reported [S1].
*   **Quality:** Preserves the overall statistical distribution of the dataset at coarser resolutions better than tensor approximation methods [S1].
*   **Complexity:** The multilevel decomposition and thresholding are computationally efficient. The use of biorthogonal filters (like 9/7) is standard and well-optimized [S1].
*   **Hardware Support:** Can be accelerated using FPGA hardware decoders in multiresolution variants [S2].

## Background
Biorthogonal Wavelet Compression is based on the principles of wavelet theory and has been applied to volume rendering to enable efficient storage and transmission of volumetric data. The use of biorthogonal 9/7 wavelets is standard in image compression (JPEG 2000) and has been adapted for 3D volumes [S1]. Variants involving octree structures and FPGA decoding have been proposed to support real-time multiresolution rendering [S2].

## Sources
[S1] State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
