# Point Cloud Filtering and Noise Removal

Also called: 3D Statistical Filter for Floater Removal, Open3D Statistical Outlier Removal, Statistical Outlier Removal

## Overview
Removes statistical outliers from a point cloud to clean noise and artifacts, specifically targeting clear floaters in unprojected point clouds [S1, unofficial][S2]. It differs from sibling methods like Farthest Point Sampling, which focuses on uniform spatial distribution, by preserving the original density of inliers while discarding points that deviate significantly from local geometric consistency [S2].

## How it works
**Inputs:** A point cloud $P$ consisting of $N$ points.
**Outputs:** A filtered point cloud $P'$ containing only inliers.

**Data Structures:**
The point cloud is represented as a list or array of 3D coordinates $(x, y, z)$. No additional metadata structures are required for the core algorithm.

**Parameters:**
*   $k$ (nb_neighbors): Number of neighboring points considered for statistical analysis. Typical value: 20 [S1, unofficial][S2].
*   $\sigma_{ratio}$ (std_ratio): Standard deviation ratio threshold. Points with a mean distance greater than $\sigma_{ratio}$ times the standard deviation of all mean distances are removed. Typical values: 1.0 to 10.0 [S1, unofficial], 2.0 [S2].

**Steps:**
1.  For each point $p_i \in P$, identify its $k$ nearest neighbors using a spatial search structure (e.g., KD-tree) [G].
2.  Calculate the mean Euclidean distance $d_i$ from $p_i$ to its $k$ neighbors:
    $$ d_i = \frac{1}{k} \sum_{j \in \text{neighbors}(i)} \| p_i - p_j \|_2 $$
3.  Compute the global mean $\mu_d$ and standard deviation $\sigma_d$ of all $d_i$ values across the entire point cloud:
    $$ \mu_d = \frac{1}{N} \sum_{i=1}^{N} d_i $$
    $$ \sigma_d = \sqrt{\frac{1}{N} \sum_{i=1}^{N} (d_i - \mu_d)^2} $$
4.  Classify point $p_i$ as an outlier if:
    $$ d_i > \mu_d + \sigma_{ratio} \cdot \sigma_d $$
5.  Construct $P'$ by retaining all points not classified as outliers.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.spatial import KDTree

def remove_statistical_outliers(points, nb_neighbors=20, std_ratio=2.0):
    """
    Remove statistical outliers from a point cloud.
    
    Args:
        points: np.ndarray of shape (N, 3) containing 3D coordinates.
        nb_neighbors: int, number of neighbors for distance calculation.
        std_ratio: float, threshold multiplier for standard deviation.
        
    Returns:
        np.ndarray: Filtered points of shape (M, 3) where M <= N.
    """
    if len(points) == 0:
        return points
        
    # Build KD-tree for efficient nearest neighbor search
    tree = KDTree(points)
    
    # Query k-nearest neighbors (include the point itself, so k+1)
    # distances shape: (N, k+1), indices shape: (N, k+1)
    distances, _ = tree.query(points, k=nb_neighbors + 1)
    
    # Exclude the point itself (distance 0) from the mean calculation
    # distances[:, 1:] contains the distances to the k neighbors
    neighbor_distances = distances[:, 1:]
    
    # Calculate mean distance to neighbors for each point
    mean_distances = np.mean(neighbor_distances, axis=1)
    
    # Calculate global mean and standard deviation of these mean distances
    global_mean = np.mean(mean_distances)
    global_std = np.std(mean_distances)
    
    # Determine threshold
    threshold = global_mean + std_ratio * global_std
    
    # Filter points: keep those below the threshold
    inliers = mean_distances < threshold
    filtered_points = points[inliers]
    
    return filtered_points
```

**Edge Cases and Pitfalls:**
*   **Small Point Clouds:** If $N \le k$, the neighbor query may fail or return insufficient neighbors. A guard should ensure $N > k$ [G].
*   **Uniform Distributions:** In perfectly uniform grids, $\sigma_d$ approaches 0, potentially causing numerical instability or removing all points if $\sigma_{ratio}$ is small. This method is best for noisy, non-uniform data [S2].
*   **Boundary Effects:** Points on the surface of a sparse cloud may have higher mean distances than internal points, leading to false positives if the cloud is not dense [G].

## Operating characteristics
*   **Time Complexity:** $O(N \log N)$ for building the KD-tree and $O(N \cdot k)$ for distance calculations, where $N$ is the number of points and $k$ is the number of neighbors.
*   **Memory Complexity:** $O(N)$ to store the point cloud and distance arrays.
*   **Strengths:** Effective at removing isolated noise points and "floaters" without altering the geometry of the main surface [S2]. Simple to implement and tune.
*   **Limitations:** Sensitive to the choice of $k$ and $\sigma_{ratio}$. May remove valid sparse points in low-density regions. Not suitable for downsampling; it only removes outliers [S1, unofficial].
*   **When to use:** After voxelization or reconstruction when isolated noise artifacts are present.
*   **When not to use:** When preserving sparse data is critical or when the point cloud is extremely dense and uniform.

## Background
The method is implemented as `remove_statistical_outlier` in the Open3D library [S1, unofficial][S2]. It is widely used in 3D computer vision pipelines, particularly in Gaussian Splatting workflows (e.g., 3DGS-to-PC) to clean point clouds generated from unprojected data [S1, unofficial][S2]. The approach relies on the statistical assumption that inliers have similar local neighborhood densities, while outliers (noise) deviate significantly [S2].

## Sources
[S1] 3DGS-to-PC: 3D Gaussian Splatting to Dense Point Clouds, https://github.com/Lewis-Stuart-11/3DGS-to-PC, fetched 2026-10-27 (unofficial)
[S2] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
