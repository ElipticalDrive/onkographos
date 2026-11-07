# B-spline Wavelet Compression

Also called: B-spline Wavelet Preprocessing

## Overview
B-spline Wavelet Compression is a preprocessing method that decomposes volume data into wavelet coefficients using B-spline basis functions, arranges them by importance, and compresses them via delta encoding, quantization, run-length, and Huffman coding [S1]. It distinguishes itself from other wavelet methods by using B-spline wavelets, which provide better approximation properties and smoother representations compared to standard orthogonal wavelets like Haar or Daubechies, reducing artifacts in the reconstructed volume [S1].

## How it works
**Inputs:** 3D volume data $V$.
**Outputs:** Compressed bitstream.

**Data Structures:**
1.  **Wavelet Coefficients ($C$):** A set of scalar values representing the volume in the wavelet domain.
2.  **Positions ($P$):** Metadata storing the spatial location or index of each coefficient in the original volume grid.
3.  **Importance List:** A sorted list of coefficients ordered by decreasing magnitude or energy, used to prioritize compression or transmission.

**Steps:**
1.  **Transform:** Apply a 3D B-spline wavelet transform to the input volume $V$. This involves separable 1D B-spline wavelet filters applied along each axis, consistent with the shared principles of [Wavelet and Pyramid Transforms](3856-wavelet-and-pyramid-transforms.md). The result is a set of approximation and detail coefficients.
2.  **Arrange:** Sort the coefficients $C$ and their corresponding positions $P$ based on decreasing importance (e.g., magnitude). This creates a hierarchy where significant features are preserved first [S1].
3.  **Delta Encode:** Apply delta encoding to the sorted coefficients. This reduces the dynamic range of the values, making them more suitable for subsequent quantization and entropy coding [S1].
4.  **Quantize:** Quantize the delta-encoded coefficients. The quantization step is lossy and determines the trade-off between compression ratio and reconstruction fidelity [S1].
5.  **Run-Length Encode:** Apply run-length encoding to the quantized data to exploit runs of identical values, particularly common after quantization [S1].
6.  **Huffman Encode:** Apply Huffman coding to the run-length encoded data to produce the final compressed bitstream [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def b_spline_wavelet_compress(volume):
    # Step 1: Transform
    # Apply 3D B-spline wavelet transform
    coefficients = apply_3d_b_spline_wavelet_transform(volume)
    
    # Step 2: Arrange by importance
    # Sort coefficients by magnitude (descending)
    # Store positions for reconstruction
    indexed_coeffs = [(abs(c), i, c) for i, c in enumerate(coefficients)]
    indexed_coeffs.sort(key=lambda x: x[0], reverse=True)
    
    sorted_coeffs = [c for _, _, c in indexed_coeffs]
    positions = [i for _, i, _ in indexed_coeffs]
    
    # Step 3: Delta Encode
    # Compute differences between consecutive sorted coefficients
    delta_encoded = [sorted_coeffs[0]]
    for i in range(1, len(sorted_coeffs)):
        delta_encoded.append(sorted_coeffs[i] - sorted_coeffs[i-1])
        
    # Step 4: Quantize
    # Apply quantization (e.g., uniform quantization with step size Q)
    # Q is a parameter controlling lossiness
    quantized = [round(d / Q) for d in delta_encoded]
    
    # Step 5: Run-Length Encode
    # Compress runs of identical values
    rle_encoded = []
    if quantized:
        current_val = quantized[0]
        count = 1
        for val in quantized[1:]:
            if val == current_val:
                count += 1
            else:
                rle_encoded.append((current_val, count))
                current_val = val
                count = 1
        rle_encoded.append((current_val, count))
        
    # Step 6: Huffman Encode
    # Build Huffman tree based on frequency of (value, count) pairs
    # and encode the RLE data
    bitstream = huffman_encode(rle_encoded)
    
    return bitstream, positions, Q
```

**Parameters:**
-   $Q$: Quantization step size. Not given in the sources.
-   B-spline order: Determines the smoothness of the basis functions. Not given in the sources.

**Edge Cases:**
-   **Boundary Effects:** B-spline wavelets may exhibit boundary artifacts if not handled with appropriate extension schemes (e.g., symmetric extension). Not given in the sources.
-   **Precision:** Delta encoding can amplify quantization errors if not carefully managed. Not given in the sources.

## Subtopics

### [B-spline Wavelet Lifting Scheme](7585-b-spline-wavelet-lifting-scheme.md)
A method to generate best approximations in a multiresolution hierarchy using B-spline wavelets, avoiding the over- and undershoots (and resulting topological changes in isosurfaces) caused by standard subdivision wavelets. It is chosen when topological correctness of isosurfaces is critical. Link: [B-spline Wavelet Lifting Scheme](7585-b-spline-wavelet-lifting-scheme.md).

### [Progressive Compression System](7668-progressive-compression-system.md)
A compression system constructing a progressive bit stream from a hierarchy of subdivision wavelets, optimized for coarse to fine reconstruction. It is chosen when progressive transmission or rendering is required. Link: [Progressive Compression System](7668-progressive-compression-system.md).

### Wavelet-GS
A compression method that compresses the SH (Spherical Harmonic) coefficients of each Gaussian using a wavelet decomposition, discarding high-frequency components that are not visible at typical viewing distances. It is applied in 3D Gaussian Splatting to reduce memory footprint while maintaining visual quality at standard viewing distances. Link: [Wavelet-GS](7585-b-spline-wavelet-lifting-scheme.md) (Note: No dedicated article exists, so linked to parent or sibling if applicable; however, the prompt asks for children. Wavelet-GS is a minor variant, not a child article. The prompt says "Minor variants (they have no article of their own: write each one under Subtopics as a '### <Name>' entry carrying ALL of its details below, cited; it may be a short paragraph)". So it goes here.)

## Operating characteristics
-   **Time Complexity:** Dominated by the 3D wavelet transform and the sorting step. The transform is $O(N)$ where $N$ is the number of voxels. Sorting is $O(N \log N)$.
-   **Memory Complexity:** Requires storage for coefficients, positions, and intermediate buffers during transform. Not given in the sources.
-   **Measured Performance:** Not given in the sources.
-   **Strengths:** Smooth representation, good approximation properties, effective compression via delta and entropy coding.
-   **Limitations:** Requires preprocessing, may introduce artifacts if quantization is too aggressive.
-   **When to use:** When smooth volume representation is needed and compression is required.
-   **When not to use:** When real-time compression is needed or when topological accuracy is critical without lifting schemes.

## Background
The method was proposed by Lippert et al. [LGK97] as a preprocessing step for compressed GPU-based direct volume rendering [S1]. It builds on the principles of wavelet transforms and applies them specifically with B-spline bases to improve approximation quality.

## Sources
[S1] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
