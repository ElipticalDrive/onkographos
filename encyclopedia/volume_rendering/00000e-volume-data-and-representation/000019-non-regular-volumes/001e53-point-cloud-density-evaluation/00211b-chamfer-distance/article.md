# Chamfer Distance
Also called: Depth Error, F-score, Geometry Metrics, Normal Error

## Overview
Chamfer Distance (CD) calculates a symmetric distance between two point clouds by averaging the nearest-neighbor distances in both directions [S1, S2]. It is primarily used to evaluate the geometric accuracy of reconstructed meshes or point clouds against ground-truth data [S3, unofficial]. Unlike sibling metrics such as Neighbor Counting or Probabilistic Occupancy Policy which assess density or occupancy, CD directly measures surface fidelity [Parent](1743-point-cloud-density-evaluation.md).

## How it works
**Inputs and Outputs:**
- Input: Two point clouds, $\chi_1$ and $\chi_2$, where each cloud is a set of 3D points.
- Output: A scalar value representing the symmetric Chamfer Distance.

**Data Structures:**
- Point clouds $\chi_1$ and $\chi_2$ are typically stored as arrays of coordinates. For efficient computation, a spatial index structure such as a KDTree is built for at least one of the clouds to enable fast nearest-neighbor queries [S4, unofficial].

**Formula:**
The Chamfer Distance is defined as [S1, S2]:

$$CD(\chi_1, \chi_2) = \frac{1}{|\chi_1|} \sum_{x \in \chi_1} \min_{y \in \chi_2} ||x - y||_2 + \frac{1}{|\chi_2|} \sum_{y \in \chi_2} \min_{x \in \chi_1} ||y - x||_2$$

Where:
- $|\chi|$ is the number of points in the cloud.
- $||\cdot||_2$ is the Euclidean distance.
- The first term averages the distance from each point in $\chi_1$ to its nearest neighbor in $\chi_2$.
- The second term averages the distance from each point in $\chi_2$ to its nearest neighbor in $\chi_1$.

**Steps:**
1. Construct a KDTree (or similar nearest-neighbor structure) for point cloud $\chi_2$ [S4, unofficial].
2. For each point $x$ in $\chi_1$, find the nearest neighbor $y$ in $\chi_2$ using the KDTree.
3. Compute the squared Euclidean distance $||x - y||_2^2$ (or Euclidean distance $||x - y||_2$) for all $x \in \chi_1$.
4. Average these distances to get the first term: $\frac{1}{|\chi_1|} \sum_{x \in \chi_1} \min_{y \in \chi_2} ||x - y||_2$.
5. Construct a KDTree for point cloud $\chi_1$.
6. For each point $y$ in $\chi_2$, find the nearest neighbor $x$ in $\chi_1$.
7. Compute the squared Euclidean distance $||y - x||_2^2$ (or Euclidean distance $||y - x||_2$) for all $y \in \chi_2$.
8. Average these distances to get the second term: $\frac{1}{|\chi_2|} \sum_{y \in \chi_2} \min_{x \in \chi_1} ||y - x||_2$.
9. Sum the two terms to obtain the final CD value.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.spatial import KDTree

def chamfer_distance(points1: np.ndarray, points2: np.ndarray) -> float:
    """
    Calculate the symmetric Chamfer Distance between two point clouds.
    
    Args:
        points1: Array of shape (N, 3) containing coordinates of the first point cloud.
        points2: Array of shape (M, 3) containing coordinates of the second point cloud.
        
    Returns:
        The symmetric Chamfer Distance.
    """
    # Build KDTree for points2 to query nearest neighbors for points1
    tree2 = KDTree(points2)
    
    # Query nearest neighbors for points1 in points2
    # k=1 returns the single nearest neighbor
    distances1_to_2, _ = tree2.query(points1, k=1)
    
    # Build KDTree for points1 to query nearest neighbors for points2
    tree1 = KDTree(points1)
    
    # Query nearest neighbors for points2 in points1
    distances2_to_1, _ = tree1.query(points2, k=1)
    
    # Calculate mean distances
    mean_dist1 = np.mean(distances1_to_2)
    mean_dist2 = np.mean(distances2_to_1)
    
    # Symmetric Chamfer Distance
    cd = mean_dist1 + mean_dist2
    
    return cd
```

**Parameters:**
- $\chi_1, \chi_2$: The two point clouds to compare [S1].
- No other specific parameters are required for the standard definition.

**Edge Cases and Pitfalls:**
- **Precision:** The metric is sensitive to outliers. A single distant point can significantly increase the distance [S1].
- **Boundaries:** The metric assumes point clouds represent surfaces. If clouds are sparse or have varying densities, the distance may not accurately reflect geometric similarity [S3, unofficial].
- **Degenerate Input:** If one point cloud is empty, the distance is undefined. Implementation should handle empty inputs by returning infinity or raising an error.

## Subtopics

### Mean Distance Computation
This variant involves computing the mean squared distance to the 3 nearest neighbors (excluding self, hence indices 1:4) and returning it as a tensor [S4, unofficial]. This approach is used in specific implementations, such as those found in the DGR repository, to evaluate point cloud quality [S4, unofficial].

### Edge Chamfer Distance
Edge Chamfer Distance (ECD) is calculated specifically for points sampled on the edges of meshes to gauge edge reconstruction quality and sharpness [S1]. It is similar to the regular Chamfer Distance but focuses on the boundary features of the geometry [S1].

## Operating characteristics
- **Time Complexity:** $O(N \log N + M \log M)$ for building KDTree structures and $O(N \log M + M \log N)$ for nearest-neighbor queries, where $N$ and $M$ are the number of points in $\chi_1$ and $\chi_2$ respectively.
- **Memory Complexity:** $O(N + M)$ to store the point clouds and KDTree structures.
- **Measured Performance:** On the ABC dataset, a CD value of 3.15 was reported for the SDC method, indicating better performance compared to other methods like MC (3.89) and NMC (3.58) [S2].
- **Strengths:** Simple to compute, symmetric, and provides a direct measure of geometric accuracy [S1, S2].
- **Limitations:** Sensitive to outliers and noise; does not account for surface normals or topology directly [S1][S3, unofficial].
- **When to Use:** When evaluating the geometric fidelity of reconstructed meshes or point clouds against ground-truth data [S3, unofficial].
- **When Not to Use:** When surface normal consistency or topological correctness is more important than raw geometric distance [S3, unofficial].

## Background
Chamfer Distance was introduced as an evaluation metric for 3D shape reconstruction tasks [S1, S2]. It is widely used in benchmarks such as ABC and Thingi10k to assess mesh reconstruction quality [S2]. The metric is also part of a broader set of geometry metrics including F-score, Depth Error, and Normal Error, which are used to evaluate reconstructed geometry in datasets like DTU and Tanks & Temples [S3, unofficial].

## Sources
[S1] Self-Supervised Dual Contouring, https://doi.org/10.1109/cvpr52733.2024.00448, fetched 2026-10-26
[S2] Self-Supervised Dual Contouring, https://inria.hal.science/hal-04590906v1, fetched 2026-10-26
[S3] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S4] wskingdom/DGR, https://github.com/wskingdom/DGR, fetched 2026-10-26 (unofficial)
