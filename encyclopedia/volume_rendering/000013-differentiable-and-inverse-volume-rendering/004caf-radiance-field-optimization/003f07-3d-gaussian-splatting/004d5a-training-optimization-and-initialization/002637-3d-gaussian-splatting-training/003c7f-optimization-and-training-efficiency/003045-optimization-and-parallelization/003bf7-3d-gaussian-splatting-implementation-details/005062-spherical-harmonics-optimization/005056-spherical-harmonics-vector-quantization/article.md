# Spherical Harmonics Vector Quantization

Also called: SH clustering, SH palette

## Overview
Spherical Harmonics Vector Quantization (SH VQ) is a compression technique for 3D Gaussian Splatting that replaces per-splat SH coefficient vectors with indices pointing to a shared palette of distinct SH vectors, significantly reducing memory footprint at the cost of minor lighting artifacts [S1, unofficial]. It sets itself apart from other optimization methods by addressing the storage bottleneck of high-dimensional view-dependent data directly through clustering, rather than altering the rendering pipeline or gradient computation [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Input:** A set of $N$ Gaussian splats, each defined by a SH coefficient vector $\mathbf{c}_i \in \mathbb{R}^{45}$ for $i=1 \dots N$. The 45 dimensions correspond to the 15 coefficients per RGB channel (excluding the 0th-order term, which is treated as base color and stored separately) [S1, unofficial].
- **Output:** A palette $\mathcal{P}$ of $K$ distinct SH vectors, and an index map $I$ where $I_i \in \{0, \dots, K-1\}$ is the index of the closest palette item for splat $i$ [S1, unofficial].

**Data Structures**
1.  **Palette ($\mathcal{P}$):** An array of size $K \times 45$. Stored as FP16 (16-bit floating point) [S1, unofficial].
2.  **Indices ($I$):** An array of size $N$. Each entry is a 16-bit unsigned integer, even if $K < 65536$ [S1, unofficial].

**Parameters**
-   **Palette Size ($K$):** Typically 4,000 to 16,000 distinct SH items [S1, unofficial].
-   **SH Dimensionality:** 45 floating-point numbers per splat [S1, unofficial].

**Algorithm Steps**
1.  **Collection:** Gather all $N$ SH coefficient vectors from the scene [S1, unofficial].
2.  **Clustering:** Cluster the $N$ vectors into $K$ clusters using a mini-batch k-means variant [S1, unofficial].
    -   The centroid of each cluster becomes an item in the palette $\mathcal{P}$ [S1, unofficial].
    -   Implementation note: A custom mini-batch k-means implementation is preferred over scikit-learn (too slow) and Faiss (external dependency) for speed and quality in this specific high-dimensional context [S1, unofficial].
3.  **Assignment:** For each splat $i$, find the palette item $p \in \mathcal{P}$ that minimizes the Euclidean distance to $\mathbf{c}_i$:
    $$ I_i = \arg \min_{j \in \{0, \dots, K-1\}} \| \mathbf{c}_i - \mathcal{P}[j] \|_2 $$
4.  **Storage:** Store the palette $\mathcal{P}$ and the index array $I$ [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def spherical_harmonics_vector_quantization(sh_data: np.ndarray, k: int) -> tuple:
    """
    Compress SH coefficients using vector quantization.
    
    Args:
        sh_data: np.ndarray of shape (N, 45), containing SH coefficients for N splats.
                 Excludes the 0th-order term.
        k: int, number of palette items (clusters).
        
    Returns:
        palette: np.ndarray of shape (k, 45), the cluster centroids.
        indices: np.ndarray of shape (N,), indices into the palette for each splat.
    """
    n_splats = sh_data.shape[0]
    
    # Step 1: Mini-batch K-Means Clustering
    # Initialize centroids randomly from the data
    indices_init = np.random.choice(n_splats, k, replace=False)
    centroids = sh_data[indices_init].copy()
    
    # Mini-batch K-Means iterations
    # Note: Standard k-means convergence criteria are used.
    # A custom implementation is recommended for performance [S1, unofficial].
    max_iters = 100
    batch_size = 1024
    
    for iteration in range(max_iters):
        # Sample a mini-batch of indices
        batch_indices = np.random.choice(n_splats, batch_size, replace=False)
        batch_data = sh_data[batch_indices]
        
        # Assign batch data to nearest centroid
        # Compute distances: (batch_size, k)
        # dist = ||x - c||^2 = ||x||^2 + ||c||^2 - 2*x.c
        sq_norms_batch = np.sum(batch_data**2, axis=1, keepdims=True)
        sq_norms_centroids = np.sum(centroids**2, axis=1, keepdims=True).T
        cross_term = batch_data @ centroids.T
        distances = sq_norms_batch + sq_norms_centroids - 2 * cross_term
        
        batch_assignments = np.argmin(distances, axis=1)
        
        # Update centroids based on assigned points in the batch
        for j in range(k):
            mask = batch_assignments == j
            if np.any(mask):
                new_centroid = np.mean(batch_data[mask], axis=0)
                centroids[j] = new_centroid
                
    palette = centroids
    
    # Step 2: Assign all splats to closest palette item
    # Compute distances for all splats
    sq_norms_all = np.sum(sh_data**2, axis=1, keepdims=True)
    sq_norms_palette = np.sum(palette**2, axis=1, keepdims=True).T
    cross_term_all = sh_data @ palette.T
    distances_all = sq_norms_all + sq_norms_palette - 2 * cross_term_all
    
    indices = np.argmin(distances_all, axis=1)
    
    return palette, indices
```

**Edge Cases and Pitfalls**
-   **Curse of Dimensionality:** Clustering millions of splats in 45 dimensions is computationally expensive and slow [S1, unofficial].
-   **Lighting Artifacts:** The quantization introduces minor lighting artifacts, though they are generally considered acceptable for the memory savings [S1, unofficial].
-   **Index Size:** Indices are stored as 16-bit integers regardless of whether the palette size $K$ requires fewer bits [S1, unofficial].

## Operating characteristics
-   **Memory Footprint:**
    -   Palette: $K \times 45 \times 2$ bytes (FP16). For $K=16,000$, this is ~1.44 MB [S1, unofficial].
    -   Indices: $N \times 2$ bytes. For $N=6.1$ million splats (e.g., 'bike' scene), this is ~12 MB [S1, unofficial].
    -   Total SH storage is drastically reduced compared to raw FP16 (350 MB for 6.1M splats) or Norm11 (187 MB) formats [S1, unofficial].
-   **Performance:**
    -   Clustering time: A custom mini-batch k-means implementation is ~3x faster than Faiss and significantly faster than scikit-learn [S1, unofficial].
    -   Rendering: Requires a lookup from the palette using the stored index, adding negligible overhead compared to direct coefficient access [S1, unofficial].
-   **Quality:**
    -   Preserves "shininess" effects effectively [S1, unofficial].
    -   'Low' preset ($K=16k$) on 'bike' scene: PSNR ~35.17-35.32, Data size ~98-93 MB (excluding other components) [S1, unofficial].
    -   'Very Low' preset ($K=4k$) on 'bike' scene: PSNR ~30.19-32.27, Data size ~79-75 MB [S1, unofficial].
-   **Strengths:** Significant reduction in data size (up to 14.9x compression for SH data in 'Low' preset) while maintaining usable quality [S1, unofficial].
-   **Limitations:** Introduces quantization artifacts; clustering is slow for very large scenes without optimized implementations [S1, unofficial].

## Background
SH VQ was introduced as a practical compression method for 3D Gaussian Splatting in a blog post by Aras Pranckevičius in September 2023 [S1, unofficial]. It draws inspiration from traditional computer graphics palette-based rendering, where colors are stored in a lookup table and pixels reference indices [S1, unofficial]. The method was implemented in a toy Unity Gaussian Splatting renderer to enable efficient real-time viewing of large scenes on consumer hardware [S1, unofficial].

## Sources
[S1] Making Gaussian Splats more smaller, https://aras-p.info/blog/2023/09/27/Making-Gaussian-Splats-more-smaller/, fetched 2026-10-26 (unofficial)
