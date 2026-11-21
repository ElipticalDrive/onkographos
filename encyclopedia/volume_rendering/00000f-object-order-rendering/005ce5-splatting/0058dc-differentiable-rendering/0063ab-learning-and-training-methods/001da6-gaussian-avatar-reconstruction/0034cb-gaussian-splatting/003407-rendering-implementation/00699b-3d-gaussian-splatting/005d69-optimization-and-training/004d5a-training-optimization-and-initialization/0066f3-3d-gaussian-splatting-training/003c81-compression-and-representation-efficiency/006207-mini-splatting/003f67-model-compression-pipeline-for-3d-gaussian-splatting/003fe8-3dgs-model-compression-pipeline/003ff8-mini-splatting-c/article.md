# Mini-Splatting-C

Also called: Ours+PP, Post-processed Compact 3D Gaussian

## Overview
Mini-Splatting-C is a storage-compression variant of the Mini-Splatting method designed for scenarios requiring minimal model footprint [S1]. It achieves significant size reduction by applying post-processing techniques to the attributes of a pre-trained Mini-Splatting model, including transform coding, quantization, and entropy encoding [S1, S2]. Unlike Mini-Splatting-D, which prioritizes rendering quality, Mini-Splatting-C optimizes for storage efficiency, demonstrating superior performance compared to other compression-oriented works applied to 3D Gaussian Splatting (3DGS) [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A pre-trained Mini-Splatting model consisting of Gaussian centers, scales, rotation quaternions, opacity, spherical harmonic (SH) coefficients, and hash grid parameters (if applicable to the specific implementation variant) [S1, S2].
*   **Output:** A compressed binary stream representing the Gaussian scene representation [S1, S2].

**Data Structures and Layout:**
The method distinguishes between attributes stored in high precision and those processed for compression [S1]:
1.  **Gaussian Centers:** Stored as `float32` (32-bit floating point) without compression [S1].
2.  **Other Attributes:** Scales, SH coefficients, opacity, and hash grid parameters undergo transform coding [S1].
3.  **Quantized Attributes:** Opacity and hash grid parameters are quantized to 8-bit integers using min-max quantization [S2].
4.  **Encoded Indices:** R-VQ (Residual Vector Quantization) indices are generated for the quantized attributes [S2].

**Steps:**
1.  **Training:** Train a standard Mini-Splatting model [S1].
2.  **Center Storage:** Store Gaussian centers as `float32` values [S1].
3.  **Transform Coding:** Apply transform coding to scales and SH coefficients [S1]. The specific transform is not detailed in the provided notes, but it is cited as [6] in the source [S1].
4.  **Quantization:** Apply 8-bit min-max quantization to opacity and hash grid parameters [S2].
    *   Formula: $q = \text{clip}(\lfloor \frac{v - v_{min}}{v_{max} - v_{min}} \times 255 + 0.5 \rfloor, 0, 255)$, where $v$ is the original value, and $v_{min}, v_{max}$ are the min/max values of the attribute [G].
5.  **Pruning:** Prune hash grid parameters with values below 0.1 [S2]. This step removes negligible contributions from the hash grid [S2].
6.  **Entropy Encoding:** Apply Huffman encoding to the quantized opacity, hash parameters, and R-VQ indices [S2].
7.  **Final Compression:** Apply standard ZIP compression to the resulting data stream [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_mini_splatting_c(model):
    # Step 1: Extract attributes from pre-trained Mini-Splatting model
    centers = model.centers  # Shape: (N, 3)
    scales = model.scales    # Shape: (N, 3)
    sh_coeffs = model.sh_coeffs # Shape: (N, K, 3)
    opacity = model.opacity  # Shape: (N,)
    hash_params = model.hash_grid_params # Shape: (M,) or similar
    
    # Step 2: Store centers as float32 (no compression)
    centers_bytes = centers.astype(np.float32).tobytes()
    
    # Step 3: Transform coding for scales and SH coefficients
    # Note: Specific transform algorithm from reference [6] is not detailed in sources
    # Assuming a placeholder for the transform coding process
    scales_transformed = apply_transform_coding(scales)
    sh_transformed = apply_transform_coding(sh_coeffs)
    
    # Step 4: Quantize opacity and hash parameters to 8-bit
    # Min-max quantization
    opacity_min = opacity.min()
    opacity_max = opacity.max()
    opacity_quantized = np.clip(
        np.round((opacity - opacity_min) / (opacity_max - opacity_min) * 255),
        0, 255
    ).astype(np.uint8)
    
    hash_min = hash_params.min()
    hash_max = hash_params.max()
    hash_quantized = np.clip(
        np.round((hash_params - hash_min) / (hash_max - hash_min) * 255),
        0, 255
    ).astype(np.uint8)
    
    # Step 5: Prune hash grid parameters below threshold
    # Note: The source mentions pruning values below 0.1. 
    # This likely refers to the original float values before quantization or 
    # specific indices in the hash grid. We assume it filters the hash_params array.
    threshold = 0.1
    valid_mask = hash_params >= threshold
    hash_params_pruned = hash_params[valid_mask]
    hash_quantized_pruned = hash_quantized[valid_mask]
    
    # Step 6: Generate R-VQ indices (Placeholder for R-VQ process)
    # The source mentions R-VQ indices are Huffman encoded.
    # Assuming R-VQ is applied to the quantized attributes.
    rvq_indices = apply_rvq(opacity_quantized, hash_quantized_pruned)
    
    # Step 7: Huffman encoding
    # Note: Standard Huffman encoding implementation
    opacity_huffman = huffman_encode(opacity_quantized)
    hash_huffman = huffman_encode(hash_quantized_pruned)
    rvq_huffman = huffman_encode(rvq_indices)
    
    # Step 8: Combine and ZIP compress
    # The source states "standard zip compression" is applied to the result of transform coding.
    # It also states Huffman is applied to quantized opacity/hash/R-VQ.
    # We combine all compressed streams.
    combined_stream = (
        centers_bytes + 
        zlib.compress(scales_transformed) + 
        zlib.compress(sh_transformed) + 
        opacity_huffman + 
        hash_huffman + 
        rvq_huffman
    )
    
    final_compressed = zlib.compress(combined_stream)
    return final_compressed
```

**Parameters:**
*   **Center Precision:** `float32` [S1].
*   **Quantization Bits:** 8-bit for opacity and hash parameters [S2].
*   **Pruning Threshold:** 0.1 for hash grid parameters [S2].

**Edge Cases and Pitfalls:**
*   **Precision Loss:** Quantization to 8-bit may introduce visual artifacts, particularly in opacity and fine details captured by hash parameters [S2].
*   **Pruning Artifacts:** Pruning hash grid parameters below 0.1 may remove valid but small contributions, potentially affecting rendering quality in sparse regions [S2].
*   **Transform Coding Dependency:** The effectiveness of the compression relies on the specific transform coding algorithm referenced as [6] in [S1], which is not detailed in the provided notes [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   **Storage:** Mini-Splatting-C achieves a storage size of 2.67 MB on the NeRF-Synthetic dataset, which is 0.04x the size of standard 3DGS [S2]. This is a reduction from the 5.55 MB of the base Mini-Splatting ("Ours") method [S2].
*   **Rendering Speed:** On the NeRF-Synthetic dataset, Mini-Splatting-C achieves 545 FPS, which is 1.52x faster than the base Mini-Splatting method (359 FPS) [S2].
*   **Training Time:** The training time for Mini-Splatting-C is not explicitly given, but the base Mini-Splatting ("Ours") takes 8m 04s on NeRF-Synthetic [S2].

**Strengths and Limitations:**
*   **Strengths:** Significant reduction in storage size compared to 3DGS and base Mini-Splatting [S1, S2]. Improved rendering speed due to reduced model size [S2]. Superior performance compared to other compression-oriented works applied to 3DGS [S1].
*   **Limitations:** Slight drop in PSNR compared to the base Mini-Splatting method (32.88 vs 33.33 on NeRF-Synthetic) [S2]. The compression pipeline adds complexity to the post-processing step [S1].

**When to Use:**
*   Use Mini-Splatting-C when storage size is the primary constraint and slight quality degradation is acceptable [S1].
*   Suitable for hardware deployment scenarios where bandwidth and memory are limited [S1].

**When Not to Use:**
*   Use Mini-Splatting-D if rendering quality is prioritized over storage size [S1].
*   Use standard 3DGS if high fidelity is required and storage/bandwidth are not constraints.

## Background
Mini-Splatting-C is part of the Mini-Splatting family, which aims to represent scenes with a constrained number of Gaussians [S1]. It was introduced in the paper "Mini-Splatting: Representing Scenes with a Constrained Number of Gaussians" [S1]. The method builds on the Mini-Splatting representation and adds post-processing compression techniques to further reduce storage size [S1]. It is designed to address the large storage requirements of 3D Gaussian Splatting [S1].

## Sources
[S1] Mini-Splatting: Representing Scenes with a Constrained Number of Gaussians, https://doi.org/10.1007/978-3-031-72980-5_10, fetched 2026-10-26
[S2] Compact 3D Gaussian Representation for Radiance Field, https://doi.org/10.1109/cvpr52733.2024.02052, fetched 2026-10-26
