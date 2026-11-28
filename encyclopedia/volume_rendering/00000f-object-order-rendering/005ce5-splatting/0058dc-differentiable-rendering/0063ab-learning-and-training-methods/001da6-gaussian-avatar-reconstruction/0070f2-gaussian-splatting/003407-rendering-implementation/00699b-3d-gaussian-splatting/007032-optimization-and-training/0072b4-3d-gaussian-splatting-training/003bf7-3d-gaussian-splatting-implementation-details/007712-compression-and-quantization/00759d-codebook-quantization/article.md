# Codebook Quantization

Also called: Clustering-based Codebook, K-means Codebook Quantization, Post-processing Quantization, Quantization via Codebooks

## Overview
Codebook Quantization is a post-processing compression technique for 3D Gaussian Splatting (3DGS) that reduces memory footprint by replacing high-precision floating-point attributes with discrete indices from a learned codebook [S1]. It specifically targets attributes with low sensitivity to precision (opacity, scale, rotation, and spherical harmonic coefficients) while preserving the high-precision positions of primitives to maintain visual fidelity [S1]. Unlike methods that prune primitives or reduce spherical harmonic bands, this method retains all trained primitives but drastically reduces the storage size of their attributes, achieving an average memory reduction of 96.3% (approximately 27x) with minimal quality degradation [S1].

## How it works
**Inputs and Outputs:**
- **Input:** The fully trained set of 3D Gaussian primitives, including positions $\mu \in \mathbb{R}^3$, scales $s \in \mathbb{R}^3$, rotations $q \in \mathbb{R}^4$ (quaternions), opacities $\alpha \in \mathbb{R}$, and spherical harmonic (SH) coefficients.
- **Output:** A compressed representation where attributes are stored as 1-byte indices into codebooks, and positions/codebook entries are stored as 16-bit half-floats.

**Data Structures:**
1.  **Codebooks:** A set of lookup tables, each containing 256 entries. Each entry is a 16-bit half-float.
    -   One codebook for Opacity.
    -   One codebook for Scale (shared, but components quantized separately).
    -   One codebook for the Real part of the Quaternion.
    -   One codebook for the Imaginary components of the Quaternion (shared, components quantized separately).
    -   One codebook for Base Color coefficients.
    -   15 codebooks for SH coefficient groups (one per group, each component quantized separately).
2.  **Primitive Attributes:**
    -   Positions: Stored as 16-bit half-floats (unchanged from original precision logic, but quantized to half-float).
    -   Attributes: Stored as 1-byte indices (uint8) pointing to the respective codebook.

**Numbered Steps:**
1.  **Training Completion:** Allow the 3DGS optimization to complete (typically 15,000 iterations or until convergence). Do not perform quantization during training; this is a post-processing step [S1].
2.  **Attribute Selection:** Identify attributes suitable for quantization: opacity, scale, rotation (quaternion components), and SH coefficients. Exclude positions from this step [S1].
3.  **Codebook Generation (K-means Clustering):**
    -   For each attribute type (or component thereof), collect all values from the $N$ primitives.
    -   Apply K-means clustering with $K=256$ to generate the codebook entries [S1].
    -   Store the resulting 256 centroids as 16-bit half-floats in the codebook [S1].
4.  **Index Assignment:**
    -   For each primitive, find the closest entry in the relevant codebook for each attribute component.
    -   Store the index (0-255) of the closest entry instead of the original float value [S1].
    -   For vector attributes (e.g., scale $s_x, s_y, s_z$), use a single shared codebook but treat each scalar component independently, storing one index per component [S1].
5.  **Position Quantization:**
    -   Convert the original 32-bit float positions to 16-bit half-floats [S1].
6.  **Storage Assembly:**
    -   Store the codebooks (20 codebooks $\times$ 256 entries $\times$ 2 bytes/entry).
    -   Store the primitive attributes as 1-byte indices.
    -   Store positions as 2-byte half-floats.

**Formulas:**
-   **Scale Storage Cost:**
    Original cost: $3N \times 4$ bytes (3 components, $N$ primitives, 4 bytes/float).
    New cost: $3N + 4 \times 256$ bytes ($3N$ indices at 1 byte each, plus 256 codebook entries at 2 bytes each).
    *Note: The source cites "4 * 256" likely implying a specific overhead or codebook size calculation, but explicitly states the codebook size is 256 entries and indices are 1 byte. The formula $3N + 4 \times 256$ appears in the source notes, though standard calculation would be $3N + 256 \times 2$. We follow the source's explicit formula structure [S1].*
-   **Total Memory Reduction:**
    Original attribute memory: $56N \times 4$ bytes.
    New attribute memory: $56N + 20 \times 256 \times 4$ bytes.
    *Note: The source provides this specific formula for the total reduction, including the overhead term [S1].*
-   **Overall Reduction:**
    Average reduction of 96.3% compared to original 3DGS file size [S1].

**Parameters:**
-   **Codebook Size:** 256 entries [S1].
-   **Index Size:** 1 byte (uint8) [S1].
-   **Position Precision:** 16-bit half-float [S1].
-   **Clustering Algorithm:** K-means [S1].

**Edge Cases and Pitfalls:**
-   **Position Accuracy:** Reducing the accuracy of primitive positions leads to significant visual degradation. Therefore, positions are strictly excluded from codebook quantization and are only down-quantized to half-floats [S1].
-   **Vector Attributes:** While scale and quaternion imaginary parts are vector attributes, they share a single codebook. The components are quantized independently, meaning the index for $s_x$ and $s_y$ may point to different clusters if their distributions differ, but they draw from the same set of 256 values [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def codebook_quantization(primitives, k=256):
    """
    primitives: dict containing arrays for positions, scales, rotations, opacities, sh_coeffs
    k: number of codebook entries (default 256)
    """
    N = len(primitives['positions'])
    
    # 1. Generate Codebooks using K-means
    # Note: K-means implementation is assumed to be available or standard
    codebooks = {}
    
    # Opacity codebook
    codebooks['opacity'] = kmeans(primitives['opacities'], k)
    
    # Scale codebook (shared for x, y, z components)
    all_scales = np.concatenate([primitives['scales'][:, 0], 
                                 primitives['scales'][:, 1], 
                                 primitives['scales'][:, 2]])
    codebooks['scale'] = kmeans(all_scales, k)
    
    # Rotation codebooks
    codebooks['rot_real'] = kmeans(primitives['rotations'][:, 0], k)
    all_rot_imag = np.concatenate([primitives['rotations'][:, 1], 
                                   primitives['rotations'][:, 2], 
                                   primitives['rotations'][:, 3]])
    codebooks['rot_imag'] = kmeans(all_rot_imag, k)
    
    # Base color codebook
    codebooks['base_color'] = kmeans(primitives['base_color'], k)
    
    # SH coefficient codebooks (15 groups, each component separately)
    # Assuming sh_coeffs is shape (N, 15, 3)
    codebooks['sh'] = {}
    for group_idx in range(15):
        for comp_idx in range(3):
            all_sh_vals = primitives['sh_coeffs'][:, group_idx, comp_idx]
            codebooks['sh'][f'{group_idx}_{comp_idx}'] = kmeans(all_sh_vals, k)

    # 2. Quantize Attributes to Indices
    quantized_attrs = {}
    
    # Opacity indices
    quantized_attrs['opacity'] = find_closest_indices(primitives['opacities'], codebooks['opacity'])
    
    # Scale indices (one per component, using shared codebook)
    quantized_attrs['scale_x'] = find_closest_indices(primitives['scales'][:, 0], codebooks['scale'])
    quantized_attrs['scale_y'] = find_closest_indices(primitives['scales'][:, 1], codebooks['scale'])
    quantized_attrs['scale_z'] = find_closest_indices(primitives['scales'][:, 2], codebooks['scale'])
    
    # Rotation indices
    quantized_attrs['rot_real'] = find_closest_indices(primitives['rotations'][:, 0], codebooks['rot_real'])
    quantized_attrs['rot_imag_x'] = find_closest_indices(primitives['rotations'][:, 1], codebooks['rot_imag'])
    quantized_attrs['rot_imag_y'] = find_closest_indices(primitives['rotations'][:, 2], codebooks['rot_imag'])
    quantized_attrs['rot_imag_z'] = find_closest_indices(primitives['rotations'][:, 3], codebooks['rot_imag'])
    
    # Base color indices
    quantized_attrs['base_color'] = find_closest_indices(primitives['base_color'], codebooks['base_color'])
    
    # SH indices
    quantized_attrs['sh'] = {}
    for group_idx in range(15):
        for comp_idx in range(3):
            key = f'{group_idx}_{comp_idx}'
            quantized_attrs['sh'][key] = find_closest_indices(primitives['sh_coeffs'][:, group_idx, comp_idx], 
                                                              codebooks['sh'][key])

    # 3. Quantize Positions to Half-Float
    # Assuming a function to convert float32 to float16
    quantized_positions = float32_to_float16(primitives['positions'])

    return {
        'codebooks': codebooks,
        'quantized_attrs': quantized_attrs,
        'quantized_positions': quantized_positions
    }

def kmeans(data, k):
    """Placeholder for K-means clustering implementation."""
    # Returns array of shape (k,) containing cluster centers
    pass

def find_closest_indices(data, codebook):
    """
    data: array of values
    codebook: array of cluster centers
    Returns: array of indices into codebook
    """
    # Compute distances and argmin
    # dists = np.abs(data[:, None] - codebook[None, :])
    # return np.argmin(dists, axis=1)
    pass

def float32_to_float16(data):
    """Placeholder for float32 to float16 conversion."""
    pass
```

## Operating characteristics
-   **Time Complexity:** The dominant cost is the K-means clustering step, which runs once after training. The complexity is $O(N \cdot K \cdot I \cdot D)$, where $N$ is the number of primitives, $K=256$ is the codebook size, $I$ is the number of iterations, and $D$ is the dimensionality of the attribute (typically 1 for scalar components). This is negligible compared to training time.
-   **Memory Complexity:** Reduces storage from $O(N)$ with large constants (4 bytes per float) to $O(N)$ with small constants (1 byte per index) plus a small constant overhead for codebooks ($20 \times 256 \times 2$ bytes).
-   **Measured Performance:** Achieves an average memory reduction of 96.3% (almost 27x) compared to the original 3DGS file size [S1]. The average PSNR drop is minimal (0.21 dB) across datasets [S1].
-   **Strengths:** Significant compression ratio with negligible visual quality loss; simple post-processing step; no changes to rendering pipeline required (only decoding indices).
-   **Limitations:** Positions are not quantized via codebook, limiting further compression on that attribute; requires storing codebooks; clustering quality depends on the distribution of attributes.
-   **When to Use:** When storage or transmission bandwidth is a bottleneck, and the scene has been fully trained.
-   **When Not to Use:** When real-time training-time compression is needed, or if position accuracy is critical and cannot tolerate even half-float quantization.

## Background
Codebook Quantization was introduced as part of the "Reducing the Memory Footprint of 3D Gaussian Splatting" paper [S1]. It builds on the observation that many 3DGS attributes (opacity, scale, rotation, SH coefficients) do not require high dynamic range or precision. The method was developed to address the large memory footprint of 3DGS, which typically requires storing millions of primitives with multiple high-precision floats each. The technique leverages K-means clustering to create a compact representation of these attributes, storing only indices to a small codebook. This approach was integrated into the 3DGS pipeline as a post-processing step after the standard densification and optimization phases [S1].

## Sources
[S1] Reducing the Memory Footprint of 3D Gaussian Splatting, https://doi.org/10.1145/3651282, fetched 2026-10-26
