# Compact3D / C3DGS / EAGLES

Also called: Compressed3D, Compression via Vector Quantization, VQ for 3DGS, Vector Quantization for 3DGS

## Overview
Compact3D (also known as C3DGS or EAGLES) is a lossy compression method for 3D Gaussian Splatting (3DGS) that reduces model storage size by replacing continuous attribute values with discrete indices into a shared codebook [S1, unofficial][S2][S3, unofficial]. It achieves approximately 4× compression without significant rendering quality degradation by applying vector quantization to specific Gaussian parameters [S2]. Unlike methods that quantize all attributes or use complex entropy coding for positions, Compact3D selectively quantizes scale and spherical harmonic (SH) coefficients while preserving the mean position and normal orientation in their original precision [S2][S3, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A trained set of $N$ 3D Gaussian primitives, where each primitive $i$ has attributes: mean position $\mu_i$, normal $n_i$, scale $s_i$, opacity $\alpha_i$, and spherical harmonic coefficients $c_i$ [S2][S3, unofficial].
- **Output:** A compressed representation consisting of:
  - Exact values for $\mu_i$ and $n_i$ [S2].
  - Indices into shared codebooks for $s_i$, $\alpha_i$, and $c_i$ [S2][S3, unofficial].
  - The shared codebooks themselves [S2].

**Data Structures:**
1.  **Codebook ($C$):** A fixed-size table of $K$ codewords. For vector attributes (e.g., scale $s \in \mathbb{R}^3$), the codebook stores vectors of the same dimension. The note specifies a shared codebook is used for vector attributes, with separate indices for each vector component [S2]. This implies either a single codebook per component dimension or a joint codebook where each entry is a vector. The text "separate indices for each vector component" suggests that for a 3D scale vector, three indices might be stored, or the codebook entries are scalars and the vector is reconstructed by combining them. However, standard VQ for vectors usually stores a vector codeword. The phrase "shared codebook... with separate indices for each vector component" is ambiguous. It likely means: For attribute $s=(s_x, s_y, s_z)$, there is one codebook for $s_x$, one for $s_y$, one for $s_z$, or one codebook where each entry is a 3D vector. Given "separate indices", it most likely means for each component of the vector attribute, an index is looked up. Let's assume the standard interpretation: A codebook $C_s$ of size $K$ contains $K$ vectors of dimension $D$ (e.g., $D=3$ for scale). Each primitive stores one index $idx_i \in [0, K-1]$. The reconstructed value is $C_s[idx_i]$. The text "separate indices for each vector component" might mean that if $s$ is 3D, we store 3 indices, each pointing to a scalar codebook? No, "shared codebook... for vector attributes" usually implies one codebook per attribute type. Let's stick to the most robust interpretation: One codebook per quantized attribute type. For scale $s$, codebook $C_s$ has $K$ entries of size 3. Each Gaussian stores one index. The "separate indices" phrase might refer to the fact that different attributes (scale, SH) have separate codebooks. Or it might mean that for a vector attribute, each component is quantized independently using the same codebook structure? The source [S2] says: "A shared codebook is used for vector attributes (e.g., scaling s), with separate indices for each vector component." This strongly suggests that for $s=(s_x, s_y, s_z)$, we store $idx_x, idx_y, idx_z$. This implies the codebook entries are scalars, or the codebook is per-component. If the codebook is per-component, then $C_{s_x}$, $C_{s_y}$, $C_{s_z}$ are separate. But it says "A shared codebook". This is contradictory. Let's look at [S3, unofficial]: "vector quantize SH and scale parameters". SH coefficients are vectors. Scale is a vector. Standard VQ for vectors uses a vector codebook. The "separate indices" likely means that for the vector attribute $s$, the index points to a vector codeword, and that codeword has components. Or, it means that the codebook is shared across all Gaussians, but indices are stored per component. Let's assume the latter for safety as it matches "separate indices for each vector component". So, for $s \in \mathbb{R}^3$, we have a codebook $C$ of size $K$ containing scalars. We store $idx_x, idx_y, idx_z$. The reconstructed $s = (C[idx_x], C[idx_y], C[idx_z])$. This is a common technique to reduce codebook size.
2.  **Primitive Data:** For each Gaussian $i$:
    - $\mu_i$: Exact float (typically FP32 or FP16) [S2][S3, unofficial].
    - $n_i$: Exact float (typically FP32 or FP16) [S2][S3, unofficial].
    - $idx_{s,i}$: Index for scale [S2].
    - $idx_{\alpha,i}$: Index for opacity [S2][S3, unofficial].
    - $idx_{c,i}$: Index for SH coefficients [S3, unofficial]. Note: SH is a vector. Does it use separate indices per coefficient? [S3, unofficial] just says "vector quantize SH". [S2] says "separate indices for each vector component" for vector attributes. It is safe to assume SH coefficients are treated as vector attributes, so separate indices per coefficient band/component are used, or a joint codebook. Given the ambiguity, we will describe the general VQ step.

**Steps:**
1.  **Select Attributes:** Identify attributes to quantize: scale $s$, opacity $\alpha$, and SH coefficients $c$. Exclude mean $\mu$ and normal $n$ [S2][S3, unofficial].
2.  **Generate Codebooks:** For each quantized attribute type (e.g., scale $s$), use k-means clustering on the values of that attribute across all $N$ Gaussians to generate a codebook $C$ of size $K$ [S2].
    - If the attribute is a vector (e.g., $s \in \mathbb{R}^3$), and "separate indices" are used, cluster each component independently or cluster the vector as a whole? The text "separate indices for each vector component" implies independent clustering per component or a scalar codebook. Let's assume scalar codebooks per component for vector attributes to minimize codebook size, as this aligns with "separate indices". So for $s$, we have codebooks $C_{s_x}, C_{s_y}, C_{s_z}$.
    - For SH, which is a vector of coefficients, apply the same logic: separate codebooks for each coefficient index? Or one codebook for the whole SH vector? [S3, unofficial] says "vector quantize SH". [S2] says "separate indices for each vector component". This implies for SH, which has multiple components (bands), we might use separate codebooks or indices. However, SH coefficients are often quantized together. Let's stick to the explicit instruction: "separate indices for each vector component". This means for a vector attribute $v$ of dimension $D$, we store $D$ indices.
3.  **Quantize:** For each Gaussian $i$ and each quantized attribute component $j$:
    - Find the index $idx_{i,j}$ in the corresponding codebook $C_j$ that minimizes the distance to the original value $v_{i,j}$ [S2].
    - $idx_{i,j} = \arg \min_{k} \| v_{i,j} - C_j[k] \|^2$ [G: Standard VQ assignment].
4.  **Store:** Save $\mu_i, n_i$ exactly. Save $idx_{i,j}$ for all other components. Save codebooks $C_j$ [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compact3d_compress(gaussians, codebook_size K):
    """
    gaussians: List of N Gaussian objects.
    Each Gaussian has attributes: mu (3,), n (3,), s (3,), alpha (1,), sh (D,)
    """
    # 1. Identify attributes to quantize
    # Exclude mu and n
    attributes_to_quantize = ['s', 'alpha', 'sh']
    
    codebooks = {}
    indices = {} # indices[g_idx][attr][comp_idx]
    
    # 2. Generate codebooks using k-means
    for attr in attributes_to_quantize:
        # Extract all values for this attribute across all Gaussians
        # If attr is a vector (like s or sh), we treat each component separately
        # based on "separate indices for each vector component"
        
        if attr == 's':
            dim = 3
        elif attr == 'alpha':
            dim = 1
        elif attr == 'sh':
            dim = len(gaussians[0].sh) # D components
        
        # Initialize codebooks for each component
        codebooks[attr] = []
        
        for comp in range(dim):
            # Collect component values
            values = [g[attr][comp] for g in gaussians]
            
            # Run k-means clustering with K clusters
            # Returns K centroids
            centroids = k_means(values, K)
            codebooks[attr].append(centroids)
            
    # 3. Quantize each Gaussian
    for i, g in enumerate(gaussians):
        indices[i] = {}
        for attr in attributes_to_quantize:
            indices[i][attr] = []
            dim = len(codebooks[attr])
            for comp in range(dim):
                val = g[attr][comp]
                cb = codebooks[attr][comp]
                # Find nearest centroid
                # Distance metric: Euclidean
                best_idx = argmin([dist(val, c) for c in cb])
                indices[i][attr].append(best_idx)
                
    # 4. Return compressed data
    # Exact mu and n are kept in original format
    # Indices and codebooks are returned
    return {
        'mu': [g.mu for g in gaussians],
        'n': [g.n for g in gaussians],
        'indices': indices,
        'codebooks': codebooks
    }

def dist(a, b):
    return (a - b) ** 2
```

**Parameters:**
- $K$: Codebook size. Not given in the sources.
- Compression ratio: Roughly 4× [S2].

**Edge Cases:**
- Quantizing $\mu$ and $n$ decreases reconstruction quality significantly, so they are excluded [S2].
- Precision: Exact values for $\mu$ and $n$ are stored, likely in FP32 or FP16 [S3, unofficial].

## Operating characteristics
- **Compression:** Provides roughly 4× compression without sacrificing rendering quality [S2].
- **Time Cost:** The time cost of the VQ process is negligible [S2].
- **Strengths:** Simple implementation, negligible training overhead, good compression ratio for editable Gaussians [S2].
- **Limitations:** Lossy compression for quantized attributes; not suitable for attributes where precision is critical (position, normal) [S2].
- **When to use:** When storage size is a concern and slight loss in attribute precision is acceptable, particularly for scale, opacity, and SH coefficients [S2][S3, unofficial].

## Background
Compact3D (also referred to as Compressed3D or C3DGS) was introduced concurrently with other vector quantization methods for 3DGS [S1, unofficial]. It is cited in the context of compressing editable 3D Gaussian Splatting models, such as in iVR-GS, where it is used to reduce the size of basic models before composition [S2]. It is also listed in surveys of Gaussian Splatting variants as a method that vector quantizes SH and scale parameters [S3, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] iVR-GS: Inverse Volume Rendering for Explorable Visualization via Editable 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3567121, fetched 2026-10-26
[S3] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
