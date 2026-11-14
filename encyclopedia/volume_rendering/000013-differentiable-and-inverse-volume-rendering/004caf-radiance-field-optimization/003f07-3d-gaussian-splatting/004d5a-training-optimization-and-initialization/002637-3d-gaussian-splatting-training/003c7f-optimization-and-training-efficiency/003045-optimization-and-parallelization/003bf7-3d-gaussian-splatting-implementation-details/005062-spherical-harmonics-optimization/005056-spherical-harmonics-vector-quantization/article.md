# Spherical Harmonics Vector Quantization

Also called: SH clustering, SH palette

## Overview
Spherical Harmonics Vector Quantization (SH VQ) is a compression technique for 3D Gaussian Splatting that replaces per-splat floating-point SH coefficients with indices pointing to a shared palette of distinct SH vectors, significantly reducing memory footprint at the cost of minor lighting artifacts [S1, unofficial]. It sets itself apart from other optimization methods by treating the high-dimensional SH data as a clustering problem rather than a per-pixel or per-splat quantization problem, allowing for shared representation across the entire scene [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** The set of all SH coefficients $\mathbf{c}_i$ for every Gaussian splat $i$ in the scene. Each splat has 45 floating-point numbers representing the SH coefficients for R, G, and B channels (excluding the 0th-order coefficient, which is treated as base color) [S1, unofficial].
- **Output:** A palette of $K$ distinct SH vectors and an index map assigning each splat to its closest palette entry [S1, unofficial].

**Data Structures:**
1.  **Palette:** An array of $K$ SH vectors. Each vector contains 45 FP16 values.
    -   Size: $K \times 45 \times 2$ bytes.
    -   Typical $K$: 4,000 to 16,000 [S1, unofficial].
2.  **Index Map:** An array of $N$ indices, one per splat.
    -   Size: $N \times 2$ bytes (stored as 16-bit unsigned integers, even if $K < 65536$) [S1, unofficial].

**Steps:**
1.  **Collect Data:** Gather all $N$ SH coefficient vectors from the scene. Each vector $\mathbf{v}_i \in \mathbb{R}^{45}$ [S1, unofficial].
2.  **Cluster:** Perform vector quantization (clustering) on the $N$ vectors to find $K$ representative centroids (the palette) [S1, unofficial].
    -   The source uses a custom mini-batch k-means implementation [S1, unofficial].
    -   The goal is to minimize the distance between each splat's SH vector and its assigned centroid [S1, unofficial].
3.  **Assign Indices:** For each splat $i$, find the index $j$ of the palette item closest to $\mathbf{v}_i$.
    -   Distance metric: Euclidean distance in the 45-dimensional space [S1, unofficial].
4.  **Store:** Save the palette and the index map [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spherical_harmonics_vector_quantization(sh_data: List[Vector45], k: int) -> Tuple[List[Vector45], List[int]]:
    """
    sh_data: List of N vectors, each of length 45 (FP32 or FP16).
    k: Number of palette entries (e.g., 4000 or 16000).
    Returns: (palette, indices)
    """
    n = len(sh_data)
    
    # Step 1: Initialize centroids using mini-batch k-means
    # Note: The source uses a custom C# implementation for speed.
    # Here we describe the logical steps of mini-batch k-means.
    
    # Initialize centroids randomly from sh_data or using k-means++
    centroids = initialize_centroids(sh_data, k)
    
    # Mini-batch k-means iterations
    for epoch in range(max_epochs):
        # Sample a mini-batch of data points
        batch_indices = random.sample(range(n), batch_size)
        batch_data = [sh_data[i] for i in batch_indices]
        
        # Update centroids based on the batch
        # For each centroid, compute the mean of the data points assigned to it in this batch
        # and update the centroid position.
        # This is a standard mini-batch k-means update step.
        centroids = update_centroids_centroids(centroids, batch_data)
        
        # Check for convergence (optional)
        if converged(centroids, old_centroids):
            break
            
    # Step 2: Assign each splat to its closest centroid
    indices = []
    for i in range(n):
        # Find the index of the centroid closest to sh_data[i]
        # Distance is Euclidean distance in 45D space
        best_idx = 0
        best_dist = infinity
        for j in range(k):
            dist = euclidean_distance(sh_data[i], centroids[j])
            if dist < best_dist:
                best_dist = dist
                best_idx = j
        indices.append(best_idx)
        
    # Step 3: Convert centroids to FP16 for storage
    palette_fp16 = [convert_to_fp16(c) for c in centroids]
    
    return palette_fp16, indices

def euclidean_distance(v1: Vector45, v2: Vector45) -> float:
    return sum((a - b) ** 2 for a, b in zip(v1, v2)) ** 0.5

def initialize_centroids(data: List[Vector45], k: int) -> List[Vector45]:
    # Simple random initialization from data points
    return random.sample(data, k)

def update_centroids_centroids(centroids: List[Vector45], batch: List[Vector45]) -> List[Vector45]:
    # Assign batch points to nearest centroid
    assignments = [[] for _ in centroids]
    for point in batch:
        best_idx = min(range(len(centroids)), key=lambda j: euclidean_distance(point, centroids[j]))
        assignments[best_idx].append(point)
        
    # Update centroids
    new_centroids = []
    for i, cluster_points in enumerate(assignments):
        if len(cluster_points) > 0:
            # Mean of points in cluster
            mean = [sum(p[d] for p in cluster_points) / len(cluster_points) for d in range(45)]
            new_centroids.append(mean)
        else:
            # Keep old centroid if cluster is empty
            new_centroids.append(centroids[i])
            
    return new_centroids
```

**Parameters:**
-   **Palette Size ($K$):** 4,000 to 16,000 [S1, unofficial].
-   **SH Dimensionality:** 45 floats per splat (15 per channel) [S1, unofficial].
-   **Index Width:** 16 bits per splat [S1, unofficial].

**Edge Cases and Pitfalls:**
-   **Curse of Dimensionality:** Clustering millions of splats in 45 dimensions is slow and may produce suboptimal results [S1, unofficial].
-   **Performance:** Standard libraries like scikit-learn are too slow; custom implementations (e.g., mini-batch k-means) are preferred for speed [S1, unofficial].
-   **Lighting Artifacts:** The method introduces minor lighting artifacts, particularly noticeable in high-frequency specular highlights, though the "shininess" effect is generally retained [S1, unofficial].

## Operating characteristics
-   **Memory Reduction:** For a scene with 6.1M splats, SH data drops from 1.1GB (FP32) or 350MB (FP16) to ~1.44MB (palette) + ~12MB (indices) = ~13.44MB [S1, unofficial].
-   **Quality:**
    -   **Low Preset (16k palette):** PSNR ~35.17-35.32 for the 'bike' scene [S1, unofficial].
    -   **Very Low Preset (4k palette):** PSNR ~30.19-32.27 for the 'bike' scene [S1, unofficial].
-   **Speed:** Custom mini-batch k-means is ~3x faster than Faiss and significantly faster than scikit-learn [S1, unofficial].
-   **Strengths:** Significant memory savings, retains general lighting structure and shininess.
-   **Limitations:** Lighting artifacts, slow clustering process for large scenes, loss of per-splat fine-grained SH detail.

## Background
SH VQ was introduced by Aras Pranckevičius in a blog post titled "Making Gaussian Splats more smaller" in September 2023 [S1, unofficial]. It was implemented in his Unity-based Gaussian Splatting viewer to enable efficient rendering of large scenes on mobile and web platforms. The technique draws inspiration from traditional image palettization and vector quantization techniques used in computer graphics history [S1, unofficial].

## Sources
[S1] Making Gaussian Splats more smaller, https://aras-p.info/blog/2023/09/27/Making-Gaussian-Splats-more-smaller/, fetched 2026-10-26 (unofficial)
