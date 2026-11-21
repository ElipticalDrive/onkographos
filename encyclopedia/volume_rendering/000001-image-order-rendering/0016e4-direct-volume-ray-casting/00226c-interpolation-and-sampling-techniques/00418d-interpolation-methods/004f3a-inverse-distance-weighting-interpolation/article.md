# Inverse Distance Weighting Interpolation

Also called: IDW, k-Nearest Neighbor Inverse Distance Weighting

## Overview
Inverse Distance Weighting (IDW) is a spatial interpolation method that estimates a value at a query point by computing a weighted average of known sample values, where weights are inversely proportional to the distance between the query point and the samples. In the context of volume rendering and point cloud processing, it is typically implemented using the $k$-nearest neighbors (k-NN) to limit computational cost and ensure local smoothness. It differs from Nearest Neighbor Sampling by providing continuous, differentiable results, and from Linear Interpolation by allowing arbitrary numbers of neighbors ($k \ge 1$) and non-grid-aligned sampling positions, though it generally produces smoother results than nearest-neighbor but less sharp detail than higher-order kernels like Lanczos [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Source Points:** A set of $m$ points with coordinates $\mathbf{xyz} \in \mathbb{R}^{m \times 3}$ and feature vectors $\mathbf{feat} \in \mathbb{R}^{m \times c}$.
*   **Query Points:** A set of $n$ points with coordinates $\mathbf{new\_xyz} \in \mathbb{R}^{n \times 3}$.
*   **Parameters:** Integer $k$ (number of neighbors), scalar $\epsilon$ (regularization constant, typically $10^{-8}$).
*   **Output:** Interpolated features $\mathbf{new\_feat} \in \mathbb{R}^{n \times c}$.

**Data Structures:**
*   Indices array $\mathbf{idx} \in \mathbb{Z}^{n \times k}$ storing the indices of the $k$ nearest neighbors in $\mathbf{xyz}$ for each query point.
*   Distances array $\mathbf{dist} \in \mathbb{R}^{n \times k}$ storing the Euclidean distances to these neighbors.

**Steps:**
1.  **Neighbor Search:** For each query point $\mathbf{q}_j \in \mathbf{new\_xyz}$, find the $k$ indices $\mathbf{idx}_{j,:}$ in $\mathbf{xyz}$ that minimize the Euclidean distance. Compute the corresponding distances $\mathbf{dist}_{j,:}$.
2.  **Weight Calculation:**
    a. Compute reciprocal distances: $r_{j,i} = \frac{1}{\mathbf{dist}_{j,i} + \epsilon}$ for $i = 1 \dots k$.
    b. Normalize weights: $w_{j,i} = \frac{r_{j,i}}{\sum_{l=1}^{k} r_{j,l}}$.
3.  **Feature Aggregation:** For each query point $j$, compute the weighted sum of features from the $k$ neighbors:
    $$ \mathbf{new\_feat}_j = \sum_{i=1}^{k} w_{j,i} \cdot \mathbf{feat}_{\mathbf{idx}_{j,i}} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def inverse_distance_weighting_interpolation(xyz, new_xyz, feat, k, epsilon=1e-8):
    """
    Interpolate features from source points to query points using k-NN IDW.
    
    Args:
        xyz: (m, 3) array of source coordinates.
        new_xyz: (n, 3) array of query coordinates.
        feat: (m, c) array of source features.
        k: int, number of neighbors.
        epsilon: float, regularization term to prevent division by zero.
        
    Returns:
        new_feat: (n, c) array of interpolated features.
    """
    n = new_xyz.shape[0]
    c = feat.shape[1]
    new_feat = np.zeros((n, c))
    
    # Step 1: Find k-nearest neighbors
    # Compute pairwise distances: (n, m)
    # Using broadcasting: new_xyz[:, None, :] - xyz[None, :, :]
    diff = new_xyz[:, np.newaxis, :] - xyz[np.newaxis, :, :]
    dist_sq = np.sum(diff ** 2, axis=2)
    
    # Get indices of k smallest distances: (n, k)
    # Note: In production, use a KD-tree or spatial hash for efficiency.
    # Here we assume a brute-force search for clarity.
    idx = np.argpartition(dist_sq, k, axis=1)[:, :k]
    
    # Gather distances for the selected neighbors: (n, k)
    dist = np.sqrt(np.take_along_axis(dist_sq, idx, axis=1))
    
    # Step 2: Compute weights
    # Reciprocal distances with epsilon regularization
    dist_recip = 1.0 / (dist + epsilon)
    
    # Normalize weights along the neighbor dimension (axis=1)
    norm = np.sum(dist_recip, axis=1, keepdims=True)
    weights = dist_recip / norm
    
    # Step 3: Aggregate features
    # Gather features for the k neighbors: (n, k, c)
    neighbor_features = np.take(feat, idx, axis=0)
    
    # Weighted sum: (n, k, c) * (n, k, 1) -> (n, k, c) -> sum over k -> (n, c)
    new_feat = np.sum(neighbor_features * weights[:, :, np.newaxis], axis=1)
    
    return new_feat
```

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If a query point coincides exactly with a source point, the distance is 0. Adding $\epsilon$ (e.g., $10^{-8}$) to the distance before inversion prevents numerical instability [S1, unofficial].
*   **Degenerate Input:** If $k > m$ (more neighbors requested than available points), the algorithm must handle the error or clamp $k$ to $m$.
*   **Boundary Effects:** Points near the boundary of the source cloud have fewer neighbors or neighbors in only one direction, potentially causing bias. This is mitigated by ensuring $k$ is small relative to local point density.

## Subtopics

### 3-Nearest Neighbors Interpolation
A specific variant where $k=3$ is used, often in the context of mesh deformation or displacement mapping. It linearly interpolates the displacements of the three nearest triangular mesh vertices using coefficients inversely related to their distances. This serves as a baseline for smoothness compared to "Closest Point" (which uses only the single nearest vertex) while being computationally cheaper than higher-order schemes. It is chosen when a balance between sharpness (nearest neighbor) and smoothness (global interpolation) is required for geometric editing tasks [S2].

## Operating characteristics
*   **Time Complexity:** $O(n \cdot m)$ for brute-force neighbor search, where $n$ is the number of query points and $m$ is the number of source points. With spatial indexing (e.g., KD-tree), this reduces to $O(n \log m)$. The weighting and aggregation steps are $O(n \cdot k \cdot c)$.
*   **Memory Complexity:** $O(m + n \cdot k)$ to store source data, query results, and neighbor indices.
*   **Strengths:** Simple to implement; produces smooth, continuous results; differentiable with respect to query positions (useful for gradient-based optimization); handles arbitrary point distributions.
*   **Limitations:** Computationally expensive for large $m$ without spatial indexing; sensitive to the choice of $k$; can introduce blurring if $k$ is too large; weights decay slowly, meaning distant points can still influence the result if not truncated (which is why k-NN is used).
*   **When to Use:** Suitable for real-time rendering where smoothness is prioritized over sharpness, or for post-processing displacement maps. Not ideal for high-frequency detail preservation compared to higher-order interpolation.

## Background
Inverse Distance Weighting is a classical geostatistical interpolation method. In computer graphics, it has been adapted for point cloud processing and neural rendering pipelines. The specific implementation using k-NN and epsilon regularization is documented in the context of 4D Gaussian Splatting and NeRF editing frameworks [S1, unofficial][S2]. It is often used as a baseline for comparing more complex interpolation schemes in geometry editing and deformation transfer tasks.

## Sources
[S1] LogosIntelligenceLab/4d-gaussian-splatting, https://github.com/LogosIntelligenceLab/4d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] NeRF-Editing: Geometry Editing of Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.01781, fetched 2026-10-26
