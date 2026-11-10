# Neighbor Counting


## Overview
Neighbor Counting is a fundamental operation in point cloud density evaluation that quantifies the local spatial density of a set of points by determining how many other points reside within a specified distance threshold of each point [S1, unofficial]. It serves as the primary mechanism for converting unstructured point clouds into density metrics, which are essential for downstream tasks such as surface reconstruction, noise filtering, and clustering [S1, unofficial]. It distinguishes itself from sibling methods like [Chamfer Distance](2446-chamfer-distance.md) by providing a local density metric rather than a global reconstruction error, and from [BrickDensityRegion Point Splatting](3956-brickdensityregion-point-splatting.md) by operating on discrete point neighborhoods rather than volumetric splatting regions.

## How it works
The method relies on spatial indexing to efficiently perform range queries. The core algorithm builds a spatial data structure from the input points and iterates through each point to count its neighbors.

**Inputs and Outputs:**
*   **Input:** A set of points $P = \{p_1, p_2, \dots, p_n\}$, where each $p_i$ is a vector in $\mathbb{R}^d$ (typically $d=3$ for volume rendering contexts). A distance threshold $\epsilon$ (epsilon).
*   **Output:** An array `neighbor_counts` of size $n$, where `neighbor_counts[i]` is the number of points in $P$ within distance $\epsilon$ of $p_i$, excluding $p_i$ itself.

**Data Structures:**
*   **KDTree:** A binary space-partitioning data structure used to store the points $P$. It supports efficient nearest-neighbor and range queries. The exact internal layout (node types, balancing criteria) is implementation-dependent but must support `query_radius` operations [S1, unofficial].

**Steps:**
1.  **Build Index:** Construct a KDTree from the input point set $P$.
2.  **Initialize:** Create an integer array `neighbor_counts` of length $n$, initialized to 0.
3.  **Query:** For each point $p_i$ in $P$:
    a. Perform a range query on the KDTree for $p_i$ with radius $\epsilon$.
    b. Let $N_i$ be the set of points returned by the query.
    c. Set `neighbor_counts[i]` = $|N_i| - 1$. The subtraction of 1 excludes the point $p_i$ itself, which is always included in the range query result [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.spatial import KDTree

def neighbor_counting(points, epsilon):
    """
    Calculate the number of neighbors within epsilon distance for each point.
    
    Args:
        points (np.ndarray): Array of shape (n, d) containing point coordinates.
        epsilon (float): The maximum distance to consider a neighbor.
        
    Returns:
        np.ndarray: Array of shape (n,) containing neighbor counts for each point.
    """
    n = points.shape[0]
    
    # Step 1: Build KDTree
    tree = KDTree(points)
    
    # Step 2: Initialize counts
    neighbor_counts = np.zeros(n, dtype=np.int32)
    
    # Step 3: Query for each point
    for i in range(n):
        # query_radius returns indices of points within epsilon
        # The point itself is always included in the result
        indices = tree.query_radius(points[i], r=epsilon)
        
        # Subtract 1 to exclude the point itself
        neighbor_counts[i] = len(indices) - 1
        
    return neighbor_counts
```

**Parameters:**
*   **epsilon ($\epsilon$):** The search radius. Typical values depend on the scale of the data. In forensic visualizations, a radius of 6.0 voxels was used for range weight calculations [S2, unofficial]. In DBSCAN contexts, it defines the neighborhood size for core points [S4, unofficial].
*   **Distance Metric:** Euclidean distance is standard, but other metrics can be used if supported by the KDTree implementation [S4, unofficial].

**Edge Cases and Pitfalls:**
*   **Self-Inclusion:** The range query typically includes the query point itself. Failing to subtract 1 results in a count that is always at least 1, which is incorrect for density estimation [S1, unofficial].
*   **Precision:** Floating-point precision errors in distance calculations can lead to points being included or excluded incorrectly at the boundary of $\epsilon$.
*   **Degenerate Input:** If $P$ is empty, the output is an empty array. If all points are identical, the count for each point is $n-1$.
*   **Performance:** For large $n$, the $O(n \log n)$ build time and $O(n \log n)$ average query time are acceptable, but worst-case query time can be $O(n)$ for highly clustered data [S1, unofficial].

## Subtopics

### [Clusterability Check](5598-clusterability-check.md)
Clusterability Check determines if a K-D tree node's grid can be clustered (optimized for rendering) by verifying if the combined QEF point count matches the sum of component point counts. It serves as a binary gate in the Neighbor Counting pipeline, preventing invalid aggregation of geometric primitives that would violate conservation laws or introduce rendering artifacts.

### [KDTree-based Neighbor Counting](6285-kdtree-based-neighbor-counting.md)
This method calculates the density of mesh vertices within each voxel of a volume grid by querying neighboring points using a KDTree and normalizing the counts. It solves the problem of quantifying point cloud density in a voxelized representation, which is a useful analysis step not always present in standard visualization libraries. It distinguishes itself from siblings like [Grid-based DBSCAN](11706-grid-based-dbscan.md) by using a fixed spherical radius query rather than density-based clustering, and from [Parzen Density Estimation](11796-parzen-density-estimation.md) by using discrete vertex counts rather than continuous kernel weighting.

### [Grid-based DBSCAN](11706-grid-based-dbscan.md)
Grid-based DBSCAN is an optimized variant of the DBSCAN clustering algorithm designed specifically for 2D spaces. It solves the problem that standard DBSCAN implementations often fail to achieve optimal time complexity or produce discrepant results compared to the original algorithm due to the limitations of general index structures like VP-trees and KD-trees. It sets itself apart from its siblings by mapping the dataset to a 2D grid and using graph connectivity to identify clusters, guaranteeing $O(n \log n)$ time complexity for 2D data, whereas [KDTree-based Neighbor Counting](6285-kdtree-based-neighbor-counting.md) relies on tree structures that may not guarantee this complexity or consistency.

### [Parzen Density Estimation](11796-parzen-density-estimation.md)
Parzen Density Estimation estimates the probability density function (pdf) of cluster data within a silhouette slice to determine cluster membership for test points, enabling the construction of a 3D pdf cube for surface reconstruction. It differs from sibling methods like KDTree-based Neighbor Counting by using a continuous Gaussian kernel interpolation to define cluster boundaries rather than discrete neighbor counts, allowing for smoother decision boundaries in the context of connecting clusters across quantized marching cubes slices.

#### Range Weight
A statistical neighborhood analysis attribute derived from intensity values that describes the neighborhood footprint in a given partial intensity range, acting as a proxy for material characteristics like homogeneity and size [S2, unofficial].
Formula: $w_r(\boldsymbol{\Phi}, N) = \frac{|N \cap V_{\boldsymbol{\Phi}}|}{|N|}$, where $N$ is an arbitrary voxel neighborhood, $V_{\boldsymbol{\Phi}}$ is the set of voxels within range $\boldsymbol{\Phi}$, and $|V|$ is the cardinality of a set $V$ [S2, unofficial].
Usage: Used in forensic visualizations to highlight metal fragments and their neighborhood, and in MR bile duct examination to isolate liver tumors [S2, unofficial].

#### Predictor Definitions
Predictor Definitions (Table 1): NOP: 0; P_X: A; P_Y: B; P_Z: D; AVG_XY: (A+B)/2; AVG_XZ: (A+D)/2; AVG_YZ: (B+D)/2; MED_XY: median(A, B, A+B-C); MED_XZ: median(A, D, A+D-E); MED_YZ: median(B, D, B+D-F); AVG3D: (A+B+D)/3. Neighbors A,B,C,D,E,F,G,H defined in Figure 2 relative to pixel X [S3].
Neighbor Locations (Figure 2): For pixel X at (slice i, row r, col c): A=(i, r, c-1), B=(i, r-1, c), C=(i, r-1, c-1), D=(i-1, r, c), E=(i-1, r, c-1), F=(i-1, r-1, c), G=(i-1, r-1, c-1), H=(i, r-1, c+1)? Note: Figure 2 shows specific relative positions. Text says 'Note: neighbors’ locations presented in Figure 2.' [S3].

#### DBSCAN
A density-based clustering algorithm that identifies arbitrary-shaped clusters and is robust to noise and data heterogeneity [S4, unofficial].
Core-point: A point p is a core-point if the number of points in its ε-neighborhood Vε(p) is greater than or equal to minPts [S4, unofficial].
Border-point: A point pa is a border-point if it is reachable from a core-point pb but |Vε(pa)| < minPts [S4, unofficial].
Noise: Points that are neither core-points nor border-points [S4, unofficial].
Step: Initialize clusterID to 0. Iterate through all points p in P and label them as UNCLASSIFIED. Iterate through all points p in P. If p is UNCLASSIFIED, call ExpandCluster(p, ε, minPts, P, clusterID). If ExpandCluster returns true, increment clusterID. If expansion proceeds, add all unclassified points retrieved from the range-query (except the initial point p) to a list of expandable points S. Iterate through S until empty. Assign all reached points to the current cluster [S4, unofficial].
Parameter: ε (eps): The search radius for range-queries. minPts: The minimum number of points required to form a dense region (core point) [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Building the KDTree takes $O(n \log n)$ on average. Querying all points takes $O(n \log n)$ on average, but can degrade to $O(n^2)$ in worst-case scenarios with highly clustered data or poor tree balancing [S1, unofficial].
*   **Memory Complexity:** $O(n)$ to store the points and the KDTree structure [S1, unofficial].
*   **Measured Performance:** Performance depends heavily on the spatial distribution of points. For uniform distributions, KDTree queries are very fast. For clustered distributions, performance degrades [S1, unofficial].
*   **Strengths:** Simple to implement, efficient for moderate-sized datasets, provides a direct measure of local density.
*   **Limitations:** Sensitive to the choice of $\epsilon$. Does not handle varying densities well without adaptive $\epsilon$. Memory intensive for very large point clouds due to KDTree construction.
*   **When to Use:** When a quick estimate of local point density is needed for preprocessing, noise filtering, or as a component in more complex algorithms like DBSCAN.
*   **When Not to Use:** For very large datasets where memory is a constraint, or when adaptive density estimation is required.

## Background
Neighbor Counting is a basic operation in computational geometry and point cloud processing. It is often used as a subroutine in more complex algorithms such as DBSCAN [S4, unofficial], surface reconstruction, and normal estimation. The use of KDTree for efficient neighbor search was popularized in the 1970s and remains a standard technique [S1, unofficial]. In the context of volume rendering and point cloud density evaluation, it provides a discrete measure of density that can be used for transfer function design and data analysis [S2, unofficial][S4, unofficial].

## Sources
[S1] wskingdom/DGR, https://github.com/wskingdom/DGR, fetched 2026-10-26 (unofficial)
[S2] Multi-Dimensional Transfer Function Design Using Sorted Histograms, https://doi.org/10.2312/vg/vg06/001-008, fetched 2026-10-26 (unofficial)
[S3] Employing New Hybrid Adaptive Wavelet-Based Transform and Histogram Packing to Improve JP3D Compression of Volumetric Medical Images, https://doi.org/10.3390/e22121385, fetched 2026-10-26
[S4] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
