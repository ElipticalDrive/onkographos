# C3DGS
Also called: Compact 3D Gaussian Splatting
## Overview
C3DGS is a compression method for 3D Gaussian Splatting that reduces storage size by applying residual vector quantization (R-VQ) to geometric attributes [S1][S2, unofficial]. It distinguishes itself from siblings like Mini-Splatting (which samples Gaussians) and CompGS (which uses different clustering strategies) by specifically targeting the redundancy in scaling and rotation vectors through a hierarchical codebook approach [S1][S2, unofficial].
## How it works
**Inputs and Outputs:**
*   **Input:** A trained set of $N$ 3D Gaussians with attributes: position $\mathbf{p}_i$, rotation $\mathbf{q}_i$, scale $\mathbf{s}_i$, opacity $\alpha_i$, and spherical harmonic coefficients $\mathbf{c}_{i, lm}$.
*   **Output:** A compressed representation using codebook indices and residuals, significantly reducing file size while maintaining visual fidelity.

**Data Structures:**
*   **Codebook:** A finite set of representative vectors (centroids) for geometric attributes (scale and rotation).
*   **Indices:** Integer indices pointing to the closest centroid in the codebook for each Gaussian.
*   **Residuals:** Small correction vectors stored to refine the approximation of the original attributes.

**Method Steps:**
1.  **Attribute Selection:** Identify geometric attributes for quantization. C3DGS selects scaling $\mathbf{s}_i$ and rotation $\mathbf{q}_i$ [S1][S2, unofficial]. Position $\mathbf{p}_i$ and opacity $\alpha_i$ are typically not quantized to avoid overlapping Gaussians and artifacts [S1][S2, unofficial].
2.  **Residual Vector Quantization (R-VQ):**
    *   Construct a hierarchical codebook. The first stage quantizes the attribute vector $\mathbf{x}_i$ (e.g., $\mathbf{s}_i$ or $\mathbf{q}_i$) to a centroid $\mathbf{c}_{1,i}$ from codebook $\mathcal{C}_1$.
    *   Compute the residual $\mathbf{r}_{1,i} = \mathbf{x}_i - \mathbf{c}_{1,i}$.
    *   Quantize the residual $\mathbf{r}_{1,i}$ using a second codebook $\mathcal{C}_2$ to find centroid $\mathbf{c}_{2,i}$.
    *   The reconstructed attribute is $\hat{\mathbf{x}}_i = \mathbf{c}_{1,i} + \mathbf{c}_{2,i}$.
    *   This process can extend to multiple stages for higher precision [S1][S2, unofficial].
3.  **Storage:** Store the codebooks and the sequence of indices for each Gaussian. Residuals may be stored explicitly or implicitly via the codebook structure.

**Parameters:**
*   Codebook sizes for each stage. Not given in the sources.
*   Number of R-VQ stages. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Quantization Error:** High compression ratios may introduce visible artifacts if the codebook is too small [S1][S2, unofficial].
*   **Attribute Correlation:** Scaling and rotation are correlated; quantizing them independently may not capture this redundancy optimally. Not given in the sources.
*   **Degenerate Input:** Gaussians with zero scale or invalid rotations must be handled before quantization. [G] Ensure valid quaternion normalization before R-VQ.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_c3dgs(gaussians, codebook1, codebook2):
    """
    Compress geometric attributes using Residual Vector Quantization.
    gaussians: List of Gaussian objects with attributes p, q, s, alpha, c_lm
    codebook1: List of centroids for first stage
    codebook2: List of centroids for second stage
    """
    compressed_data = []
    
    for g in gaussians:
        # Select geometric attributes: scale s and rotation q
        # Note: Position p and opacity alpha are not quantized [S1][S2, unofficial]
        
        # Quantize Scale s
        idx1_s = find_closest_centroid(g.s, codebook1)
        residual_s = g.s - codebook1[idx1_s]
        idx2_s = find_closest_centroid(residual_s, codebook2)
        reconstructed_s = codebook1[idx1_s] + codebook2[idx2_s]
        
        # Quantize Rotation q (Quaternion)
        # Note: Quaternion normalization is required [G]
        q_normalized = normalize_quaternion(g.q)
        idx1_q = find_closest_centroid(q_normalized, codebook1)
        residual_q = q_normalized - codebook1[idx1_q]
        idx2_q = find_closest_centroid(residual_q, codebook2)
        reconstructed_q = codebook1[idx1_q] + codebook2[idx2_q]
        
        # Store indices and non-quantized attributes
        compressed_data.append({
            'p': g.p,
            'alpha': g.alpha,
            'c_lm': g.c_lm,
            'idx_s': [idx1_s, idx2_s],
            'idx_q': [idx1_q, idx2_q]
        })
        
    return compressed_data, codebook1, codebook2

def find_closest_centroid(vector, codebook):
    """Find the index of the closest centroid in the codebook."""
    min_dist = float('inf')
    closest_idx = -1
    for i, centroid in enumerate(codebook):
        dist = euclidean_distance(vector, centroid)
        if dist < min_dist:
            min_dist = dist
            closest_idx = i
    return closest_idx

def euclidean_distance(v1, v2):
    """Calculate Euclidean distance between two vectors."""
    return sum((a - b) ** 2 for a, b in zip(v1, v2)) ** 0.5

def normalize_quaternion(q):
    """Normalize a quaternion to unit length."""
    norm = sum(x ** 2 for x in q) ** 0.5
    return [x / norm for x in q]
```
## Operating characteristics
*   **Size:** On the MipNeRF360 dataset, C3DGS achieves a size of 48.8 MB [S1].
*   **Quality:** SSIM 0.798, PSNR 27.08, LPIPS 0.247 on MipNeRF360 [S1].
*   **Comparison:** Smaller than 3DGS (750 MB) and Compact3D (68 MB), but larger than LightGaussian (42.48 MB) and SASCGS (28.80 MB) [S1].
*   **Strengths:** Effective reduction in geometric attribute size using R-VQ [S1][S2, unofficial].
*   **Limitations:** Does not quantize position or opacity, limiting further compression [S1][S2, unofficial].
## Background
C3DGS was introduced as a method to reduce computational resources and storage size for 3D Gaussian Splatting [S1]. It builds on the principle that vector quantization is a traditional compression method in signal processing [S1][S2, unofficial]. The method specifically targets geometric attributes (scaling and rotation) using residual vector quantization [S1][S2, unofficial].
## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
