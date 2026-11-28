# K-means Parameter Compression

Also called: K-means clustering compression, torchpq K-means

## Overview
K-means Parameter Compression reduces the memory footprint of Gaussian Splatting parameters by clustering high-dimensional feature vectors into a fixed number of centroids using K-means, then storing only the cluster labels and quantized centroid values [S2, unofficial]. It operates as a post-training compression step, requiring no model retraining [G].

## How it works
**Inputs:**
- `params`: A tensor of Gaussian Splatting parameters to compress.
- `n_clusters`: Number of K-means clusters (default: 65536) [S1, S2, unofficial].
- `quantization`: Number of bits for centroid quantization (default: 6) [S1, S2, unofficial].

**Outputs:**
- `centroids_quant`: Quantized centroid values stored as `uint8`.
- `labels`: Cluster assignments for each parameter vector stored as `uint16`.
- `metadata`: Dictionary containing `shape`, `dtype`, `mins`, `maxs`, and `quantization` bits [S2, unofficial].

**Data Structures:**
- `centroids_quant`: Array of shape `(n_features, n_clusters)` of type `uint8` [S2, unofficial].
- `labels`: Array of shape `(n_params,)` of type `uint16` [S2, unofficial].
- `metadata`: Dictionary with keys `shape` (tuple), `dtype` (string), `mins` (array), `maxs` (array), `quantization` (int) [S2, unofficial].

**Steps:**
1. Initialize K-means object with `n_clusters`, distance metric `manhattan`, and verbose flag [S2, unofficial].
2. Reshape `params` to `(params.shape[0], -1)`, permute to `(-1, params.shape[0])`, and make contiguous to form input tensor `x` [S2, unofficial].
3. Fit K-means on `x` to obtain labels [S2, unofficial].
4. Detach labels, move to CPU, and convert to numpy array [S2, unofficial].
5. Extract centroids from the K-means object, permute to `(-1, n_clusters)`, and convert to numpy array [S2, unofficial].
6. Compute `mins` and `maxs` of the centroids tensor along the feature axis [S2, unofficial].
7. Normalize centroids to `[0, 1]` range:
   $$ \text{centroids\_norm} = \frac{\text{centroids} - \text{mins}}{\text{maxs} - \text{mins}} $$
   [S2, unofficial].
8. Quantize normalized centroids to `uint8` using the formula:
   $$ \text{centroids\_quant} = \left\lfloor \frac{\text{centroids} - \text{mins}}{\text{maxs} - \text{mins}} \times (2^{\text{quantization}} - 1) \right\rceil $$
   Cast to `np.uint8` [S1, unofficial].
9. Convert labels to `uint16` [S2, unofficial].
10. Save `centroids_quant` and `labels` to an NPZ file using `np.savez_compressed` [S2, unofficial].
11. Return metadata dictionary containing `shape`, `dtype`, `mins`, `maxs`, and `quantization` bits [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch
from torchpq.clustering import KMeans

def kmeans_compress(params, n_clusters=65536, quantization=6):
    # Step 2: Prepare input tensor
    x = params.reshape(params.shape[0], -1).permute(1, 0).contiguous()
    
    # Step 1 & 3: Initialize and fit K-means
    kmeans = KMeans(n_clusters=n_clusters, distance='manhattan', verbose=False)
    labels_tensor = kmeans.fit(x)
    
    # Step 4: Process labels
    labels = labels_tensor.detach().cpu().numpy().astype(np.uint16)
    
    # Step 5: Extract centroids
    centroids = kmeans.centroids.detach().cpu().numpy()
    centroids = centroids.T  # Transpose to (n_features, n_clusters)
    
    # Step 6: Compute mins and maxs
    mins = centroids.min(axis=1)
    maxs = centroids.max(axis=1)
    
    # Step 7: Normalize
    denom = maxs - mins
    denom[denom == 0] = 1.0  # Guard against division by zero [G]
    centroids_norm = (centroids - mins.reshape(-1, 1)) / denom.reshape(-1, 1)
    
    # Step 8: Quantize
    scale = 2**quantization - 1
    centroids_quant = np.round(centroids_norm * scale).astype(np.uint8)
    
    # Step 10 & 11: Save and return metadata
    metadata = {
        'shape': params.shape,
        'dtype': str(params.dtype),
        'mins': mins,
        'maxs': maxs,
        'quantization': quantization
    }
    
    # np.savez_compressed('compressed.npz', centroids_quant=centroids_quant, labels=labels)
    
    return centroids_quant, labels, metadata
```

**Edge Cases and Pitfalls:**
- Division by zero in normalization if `maxs == mins`. Handled by clamping the denominator to 1.0 [G].
- Precision loss during quantization. Using `uint8` limits centroid precision, though the default 6-bit setting leaves the upper 2 bits unused [S2, unofficial].
- Memory usage during K-means fitting on large tensors. The method reshapes and permutes, which may create intermediate copies [G].

## Subtopics

### [Inference-Time Clustering](17336-inference-time-clustering.md)
Groups geometry tokens into clusters via k-means and merges each cluster into a single scene token, reducing the Gaussian budget while preserving cross-view consistency. It allows adjustable compression at inference without retraining, distinguishing it from methods that require fixed budgets or retraining for different compression levels. Unlike nearest-seed assignment, it uses iterative refinement to improve quality, though the gain is marginal.

## Operating characteristics
- **Time Complexity:** Dominated by iterative K-means convergence [G]. The number of iterations depends on convergence criteria [G].
- **Memory Complexity:** Stores `n_clusters * n_features` centroids and `n_params` labels [G]. Quantization stores centroids in `uint8`, though the default 6-bit precision leaves the upper 2 bits unused [S2, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Reduces memory footprint by storing only labels and quantized centroids [S2, unofficial].
- **Limitations:** Quantization introduces reconstruction error [G].
- **When to Use:** When memory is constrained for Gaussian Splatting parameters [S2, unofficial].
- **When Not to Use:** When high fidelity is critical and memory is not a constraint [G].

## Background
K-means Parameter Compression is used in Gaussian Splatting parameter compression [S2, unofficial]. It was introduced in the `splatad` repository [S2, unofficial]. The implementation leverages the `torchpq` library for K-means clustering [S2, unofficial].

## Sources
[S1] SNU-ARC/GS-Scale, https://github.com/SNU-ARC/GS-Scale, fetched 2026-10-27 (unofficial)
[S2] carlinds/splatad, https://github.com/carlinds/splatad, fetched 2026-10-27 (unofficial)
