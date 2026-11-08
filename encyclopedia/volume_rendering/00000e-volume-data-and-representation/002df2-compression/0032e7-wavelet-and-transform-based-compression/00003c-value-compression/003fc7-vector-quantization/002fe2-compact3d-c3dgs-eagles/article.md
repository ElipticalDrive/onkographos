# Compact3D / C3DGS / EAGLES

Also called: Compressed3D, Compression via Vector Quantization, VQ for 3DGS, Vector Quantization for 3DGS

## Overview
Compact3D (also referred to as Compressed3D, C3DGS, or EAGLES) is a lossy compression method for 3D Gaussian Splatting (3DGS) that applies vector quantization (VQ) to Gaussian attributes to reduce model storage size [S1, S4, unofficial]. It clusters similar attribute values into shared codewords, replacing exact parameter storage with indices into a fixed-size codebook [S1, unofficial][S2]. Unlike sibling methods that may quantize all attributes or use context-aware entropy coding, Compact3D specifically excludes mean ($\mu$) and normal ($n$) positions from quantization to preserve reconstruction quality, focusing instead on spherical harmonics (SH) and scale parameters [S2][S4, unofficial]. It is often deployed as the final compression stage after iterative pruning and SH degree reduction [S3].

## How it works

### Inputs and Outputs
*   **Input:** A trained set of 3D Gaussian primitives, each defined by mean $\mu$, covariance (derived from scale $s$ and rotation $q$), opacity $\alpha$, and spherical harmonics coefficients $c$.
*   **Output:** A compressed representation consisting of:
    1.  Exact values for mean $\mu$ and normal $n$ (or rotation $q$).
    2.  Shared codebooks for other attributes (scale $s$, SH coefficients, opacity $\alpha$).
    3.  Indices pointing to the nearest codeword in the respective codebooks for each primitive.

### Data Structures
1.  **Codebook ($C$):** A fixed-size table of $K$ codewords.
    *   For scalar attributes (e.g., opacity), $C$ is a vector of size $K$.
    *   For vector attributes (e.g., scale $s \in \mathbb{R}^3$), $C$ is a matrix of shape $K \times D$, where $D$ is the vector dimension.
2.  **Primitive Index ($I$):** For each Gaussian primitive $i$, an integer index $I_i \in \{0, \dots, K-1\}$ referencing the selected codeword.
3.  **Exact Attributes:** Stored in standard floating-point format (e.g., FP32 or FP16) for $\mu$ and $n$.

### Steps
1.  **Attribute Selection:** Identify attributes suitable for quantization. Typically, scale $s$ and SH coefficients are quantized. Mean $\mu$ and normal $n$ (or rotation $q$) are excluded to prevent significant quality degradation [S2][S4, unofficial].
2.  **Codebook Generation:** For each selected attribute, generate a shared codebook using k-means clustering across all Gaussian primitives [S2].
    *   Let $V$ be the set of all values for a specific attribute across all primitives.
    *   Run k-means with $K$ clusters on $V$ to obtain centroids $\{c_1, \dots, c_K\}$.
    *   These centroids form the codebook $C$.
3.  **Index Assignment:** For each primitive $i$ and each quantized attribute:
    *   Find the index $I_i$ of the codeword $c_{I_i}$ in $C$ that minimizes the distance to the original attribute value $v_i$.
    *   For vector attributes, the codebook is shared, but separate indices are stored for each vector component if the implementation treats components independently, or a single index if the codeword is a full vector [S2]. The source notes specify: "A shared codebook is used for vector attributes (e.g., scaling s), with separate indices for each vector component" [S2]. This implies that for a 3D scale vector, three separate codebooks (or one codebook with 3D codewords, but indexed per component?) are used. The text says "separate indices for each vector component," which usually implies scalar codebooks per component or a vector codebook where the index is shared but the value is reconstructed. However, "separate indices" suggests $I_{i,x}, I_{i,y}, I_{i,z}$ from potentially shared or separate codebooks. Given "shared codebook... with separate indices," it likely means one codebook per attribute type, but for vectors, you might store 3 indices if the codebook is scalar, or 1 index if the codebook is vector. The text says "separate indices for each vector component," which strongly suggests scalar codebooks per component or a mechanism where each component is quantized independently against a shared distribution. Let's assume scalar codebooks per component for clarity, or a vector codebook where the index is stored per component? No, "separate indices" implies multiple indices per primitive per attribute. If the codebook is shared, it might be one codebook for all x-components, one for all y, etc. Or one codebook for the whole vector, but the text says "separate indices". Let's stick to the literal: "separate indices for each vector component" [S2].
4.  **Storage:** Save the exact $\mu$ and $n$, the codebooks $C$, and the indices $I$.

### Formulas
*   **Distance Metric:** The nearest codeword is found by minimizing the Euclidean distance:
    $$ I_i = \arg \min_{k \in \{1, \dots, K\}} \| v_i - c_k \|^2 $$
*   **Compression Ratio:** The size reduction is approximately $4\times$ for VQ alone [S2], or up to $3.7\times$ additional reduction when combined with pruning and SH reduction [S3].

### Parameters
*   **Codebook Size ($K$):** Not explicitly given in the notes, but implied to be small enough to achieve high compression. Typical VQ sizes are powers of 2 (e.g., 256, 1024).
*   **Attributes Quantized:** Scale $s$, SH coefficients, Opacity $\alpha$ [S2][S4, unofficial].
*   **Attributes Excluded:** Mean $\mu$, Normal $n$ [S2].

### Edge Cases and Pitfalls
*   **Quality Degradation:** Quantizing mean $\mu$ or normal $n$ notably decreases reconstruction quality [S2].
*   **Precision:** The method relies on the accuracy of the k-means clustering. Poor initialization or insufficient $K$ can lead to suboptimal codewords.
*   **Vector Components:** Handling vector attributes requires care. The source specifies separate indices for each component, which may increase storage overhead if not managed carefully [S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compact3d_compress(gaussians, codebook_size):
    """
    Compress 3D Gaussian Splatting primitives using Vector Quantization.
    
    Args:
        gaussians: List of Gaussian primitives. Each primitive has attributes:
                   mu (mean, 3D), n (normal/rotation, 3D/4D), s (scale, 3D),
                   alpha (opacity, scalar), sh (SH coefficients, vector).
        codebook_size: Number of codewords K for each codebook.
    
    Returns:
        compressed_data: Dictionary containing:
                         - exact_mu: Array of exact mean values.
                         - exact_n: Array of exact normal/rotation values.
                         - codebooks: Dict mapping attribute name to codebook array.
                         - indices: Dict mapping attribute name to array of indices.
    """
    compressed_data = {}
    
    # 1. Store exact values for non-quantized attributes
    compressed_data['exact_mu'] = [g.mu for g in gaussians]
    compressed_data['exact_n'] = [g.n for g in gaussians]
    
    # 2. Identify attributes to quantize
    # Based on sources: scale s, SH coefficients, opacity alpha are quantized.
    # Mean mu and normal n are excluded.
    attributes_to_quantize = ['s', 'alpha', 'sh']
    
    for attr in attributes_to_quantize:
        # 3. Extract all values for this attribute across all primitives
        values = [getattr(g, attr) for g in gaussians]
        
        # 4. Generate codebook using k-means clustering
        # Note: For vector attributes (s, sh), k-means is applied to the flattened vectors
        # or per-component if specified. Source says "separate indices for each vector component".
        # This implies we might need to handle vectors carefully.
        # If 's' is 3D, we might create 3 codebooks (one per dim) or one codebook for 3D vectors.
        # "separate indices for each vector component" suggests per-component codebooks.
        
        if isinstance(values[0], (list, tuple)) or hasattr(values[0], '__len__'):
            # Vector attribute
            dim = len(values[0])
            codebooks = []
            indices = []
            for d in range(dim):
                # Extract d-th component for all primitives
                component_values = [v[d] for v in values]
                # Run k-means on component_values
                cb, idx = kmeans(component_values, codebook_size)
                codebooks.append(cb)
                indices.append(idx)
            compressed_data['codebooks_' + attr] = codebooks
            compressed_data['indices_' + attr] = indices
        else:
            # Scalar attribute
            cb, idx = kmeans(values, codebook_size)
            compressed_data['codebooks_' + attr] = cb
            compressed_data['indices_' + attr] = idx
            
    return compressed_data

def kmeans(values, k):
    """
    Simple k-means clustering implementation.
    Returns centroids (codebook) and indices.
    """
    # Initialize centroids randomly from values
    centroids = random.sample(values, k)
    indices = [0] * len(values)
    
    for _ in range(100): # Max iterations
        # Assign each value to nearest centroid
        new_indices = []
        for v in values:
            dists = [euclidean_distance(v, c) for c in centroids]
            new_indices.append(argmin(dists))
        
        # Update centroids
        new_centroids = []
        for i in range(k):
            cluster_vals = [v for v, idx in zip(values, new_indices) if idx == i]
            if cluster_vals:
                new_centroids.append(mean(cluster_vals))
            else:
                new_centroids.append(centroids[i]) # Keep old if empty
        
        if new_centroids == centroids:
            break
        centroids = new_centroids
        indices = new_indices
        
    return centroids, indices

def euclidean_distance(a, b):
    return sum((x - y) ** 2 for x, y in zip(a, b)) ** 0.5

def argmin(lst):
    return min(range(len(lst)), key=lambda i: lst[i])

def mean(lst):
    return sum(lst) / len(lst)
```

## Subtopics

### Hierarchical Vector Quantization
Uses a tree-like structure for codebooks to allow progressive refinement. Chose when variable bitrate is needed. [Link](827-hierarchical-vector-quantization.md)

### LBG Algorithm
A specific algorithm for generating VQ codebooks (Linde-Buzo-Gray). Chose for standard codebook generation. [Link](840-lbg-algorithm.md)

### Vector Quantization for Volumetric Radiance Fields
Applies VQ to NeRF-like fields. Chose for continuous fields rather than discrete Gaussians. [Link](1870-vector-quantization-for-volumetric-radiance-fields.md)

### Fortune's Palette Compression
Uses palette-based compression for colors. Chose for color-specific optimization. [Link](4988-fortune-s-palette-compression.md)

### Codebook Sharing
Shares codebooks across different attributes or scenes. Chose to reduce redundancy. [Link](7563-codebook-sharing.md)

### Brood-based Vector Quantization
Uses a generative approach for codebooks. Chose for high-fidelity reconstruction. [Link](7595-brood-based-vector-quantization.md)

## Operating characteristics
*   **Compression Ratio:** VQ alone provides roughly $4\times$ compression [S2]. When combined with iterative pruning and SH reduction, total compression reaches $51.6\times$ [S3].
*   **Quality Impact:** VQ alone causes negligible quality loss [S2]. Combined with pruning and SH reduction, it adds a $0.48$ dB average PSNR drop [S3]. Total PSNR drop across datasets is $0.743$ dB [S3].
*   **Performance:** On edge hardware (TSMC 28 nm), the compressed model achieves 129 FPS at 1080p with 0.219 W power consumption [S3].
*   **Strengths:** High compression ratio with minimal quality loss; suitable for edge deployment.
*   **Limitations:** Requires careful selection of attributes to quantize; excludes $\mu$ and $n$ to maintain quality.
*   **When to Use:** Edge devices with limited memory and power; scenarios requiring high compression ratios.
*   **When Not to Use:** High-fidelity applications where minimal quality loss is critical; scenes with high geometric complexity where $\mu$ quantization might be beneficial (though not recommended by sources).

## Background
Compact3D (Compressed3D, C3DGS, EAGLES) was introduced concurrently with other 3DGS compression methods [S1, unofficial]. It leverages vector quantization to cluster Gaussian parameters, building on the anchor-based representation and context-aware approaches [S1, unofficial]. It is designed for hardware-optimized 3DGS rendering, targeting low-power edge deployment [S3]. The method is part of a broader trend of compressing 3DGS for efficient storage and transmission [S4, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] iVR-GS: Inverse Volume Rendering for Explorable Visualization via Editable 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3567121, fetched 2026-10-26
[S3] A 129 FPS Full HD Real-Time Accelerator for 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3683714, fetched 2026-10-27
[S4] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
