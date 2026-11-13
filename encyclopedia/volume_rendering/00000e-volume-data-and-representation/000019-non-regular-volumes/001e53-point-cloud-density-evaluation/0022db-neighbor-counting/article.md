# Neighbor Counting


## Overview
Neighbor Counting is a fundamental technique in volume data and representation that estimates the local density of a point cloud or voxel grid by determining the number of neighboring points within a specified radius for each data point [S1, unofficial]. It solves the problem of converting discrete spatial data into a scalar density field, which serves as a prerequisite for clustering, noise reduction, and density-based visualization [S1, unofficial]. It distinguishes itself from its siblings by providing a raw, discrete count of spatial proximity, whereas [Chamfer Distance](2446-chamfer-distance.md) measures pairwise distances between sets, [BrickDensityRegion Point Splatting](3956-brickdensityregion-point-splatting.md) projects density onto screen space, and [Probabilistic Occupancy Policy](5043-probabilistic-occupancy-policy.md) uses learned probabilities rather than geometric counts [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ points $P = \{p_1, p_2, \dots, p_N\}$ in $\mathbb{R}^3$, and a distance threshold $\epsilon$ (radius).
*   **Output:** An array `neighbor_counts` of size $N$, where `neighbor_counts[i]` is the number of points in $P$ within distance $\epsilon$ of $p_i$, excluding $p_i$ itself.

**Data Structures:**
*   **KDTree:** A binary space-partitioning data structure storing the input points $P$. Each node contains a point index, a splitting axis, and left/right child references.
*   **neighbor_counts:** A 1D array of integers (or floats) of length $N$.

**Steps:**
1.  **Build Index:** Construct a KDTree from the input point set $P$. This allows for efficient range queries [S1, unofficial].
2.  **Query Neighbors:** For each point $p_i \in P$:
    a.  Perform a range query on the KDTree with center $p_i$ and radius $\epsilon$.
    b.  Retrieve the set of indices $R_i = \{j \mid \|p_j - p_i\| \le \epsilon\}$.
    c.  Calculate the count $c_i = |R_i| - 1$. The subtraction of 1 excludes the point $p_i$ itself, which is always within distance 0 [S1, unofficial].
3.  **Store Result:** Assign `neighbor_counts[i] = c_i`.

**Formulas:**
The neighbor count $c_i$ for point $p_i$ is defined as:
$$ c_i = \left| \{ j \in \{1, \dots, N\} : \|p_j - p_i\|_2 \le \epsilon \} \right| - 1 $$
where $\| \cdot \|_2$ is the Euclidean distance metric [S1, unofficial].

**Parameters:**
*   $\epsilon$ (distance_threshold): The radius of the neighborhood. Typical values depend on the scale of the data; for normalized unit cubes, values like $0.01$ to $0.1$ are common. Not given in the sources.
*   Metric: Euclidean distance is implied by the standard KDTree usage [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Self-Counting:** The query radius includes the point itself. The algorithm must explicitly subtract 1 from the raw query result to get the *neighbor* count [S1, unofficial].
*   **Boundary Effects:** Points near the boundary of the domain have fewer potential neighbors. This is inherent to the definition and not an error, but affects density interpretation.
*   **Precision:** Floating-point comparisons for $\|p_j - p_i\| \le \epsilon$ should use a small epsilon for tolerance if exact equality is not required, though standard KDTree implementations handle this internally.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.spatial import KDTree

def neighbor_counting(points: np.ndarray, distance_threshold: float) -> np.ndarray:
    """
    Calculate the number of neighbors for each point within a given radius.
    
    Args:
        points: Array of shape (N, 3) containing the coordinates of N points.
        distance_threshold: The radius epsilon for neighborhood queries.
        
    Returns:
        Array of shape (N,) containing the neighbor count for each point.
    """
    # Step 1: Build KDTree
    tree = KDTree(points)
    
    # Step 2: Query neighbors for each point
    # query_radius returns a list of lists, where each inner list contains indices
    # of points within distance_threshold of the corresponding query point.
    neighbor_indices = tree.query_radius(points, r=distance_threshold)
    
    # Step 3: Calculate counts, excluding self
    # The count is the length of the neighbor list minus 1 (for the point itself)
    counts = np.array([len(indices) - 1 for indices in neighbor_indices])
    
    return counts
```

## Subtopics

### [Clusterability Check](5598-clusterability-check.md)
Determines if a K-D tree node's grid can be clustered by verifying if the combined QEF point count matches the sum of component point counts. It serves as a binary gate in the Neighbor Counting pipeline, preventing invalid aggregation of geometric primitives that would violate conservation laws or introduce rendering artifacts.

### [KDTree-based Neighbor Counting](6285-kdtree-based-neighbor-counting.md)
Calculates the density of mesh vertices within each voxel of a volume grid by querying neighboring points using a KDTree and normalizing the resulting counts. It distinguishes itself from its siblings by using a spatial index (KDTree) for efficient neighbor retrieval rather than grid-based binning [11706-grid-based-dbscan.md] or kernel smoothing [13917-kernel-density-estimation-kde.md], and from [5598-clusterability-check.md] by focusing on scalar density estimation rather than cluster validity.

### [Grid-based DBSCAN](11706-grid-based-dbscan.md)
An optimized variant of DBSCAN for 2D spaces that maps the dataset to a 2D grid and uses graph connectivity to identify clusters, guaranteeing $O(n \log n)$ time complexity for 2D data, whereas [KDTree-based Neighbor Counting](6285-kdtree-based-neighbor-counting.md) relies on tree structures that may not guarantee this complexity or consistency.

### [DBSCAN](11722-dbscan.md)
A density-based clustering algorithm that identifies arbitrary-shaped clusters in point cloud data while robustly handling noise and data heterogeneity. It distinguishes itself from centroid-based methods like K-Means by its ability to detect non-convex shapes and its explicit classification of outliers as noise.

### [Parzen Density Estimation](11796-parzen-density-estimation.md)
Estimates the probability density function (pdf) of cluster data within a silhouette slice to determine cluster membership for test points. It differs from sibling methods like KDTree-based Neighbor Counting by using a continuous Gaussian kernel interpolation to define cluster boundaries rather than discrete neighbor counts, allowing for smoother decision boundaries in the context of connecting clusters across quantized marching cubes slices.

### Range Weight
A statistical neighborhood analysis attribute derived from intensity values that describes the neighborhood footprint in a given partial intensity range, acting as a proxy for material characteristics like homogeneity and size [S2, unofficial]. It is calculated as $w_r(\boldsymbol{\Phi}, N) = \frac{|N \cap V_{\boldsymbol{\Phi}}|}{|N|}$, where $N$ is an arbitrary voxel neighborhood, $V_{\boldsymbol{\Phi}}$ is the set of voxels within range $\boldsymbol{\Phi}$, and $|V|$ is the cardinality of a set $V$ [S2, unofficial]. It is used in forensic visualizations to highlight metal fragments and their neighborhood, and in MR bile duct examination to isolate liver tumors [S2, unofficial].

### Predictor Definitions
Defines a set of candidate predictors for adaptive wavelet-based transform construction in volumetric image compression, based on the values of neighboring pixels in a 3D grid [S3]. The layout includes NOP (0), P_X (A), P_Y (B), P_Z (D), AVG_XY ((A+B)/2), AVG_XZ ((A+D)/2), AVG_YZ ((B+D)/2), MED_XY (median(A, B, A+B-C)), MED_XZ (median(A, D, A+D-E)), MED_YZ (median(B, D, B+D-F)), and AVG3D ((A+B+D)/3). The neighbors A, B, C, D, E, F, G, H are defined relative to a target pixel X in a 3D volume slice structure, where A=(i, r, c-1), B=(i, r-1, c), C=(i, r-1, c-1), D=(i-1, r, c), E=(i-1, r, c-1), F=(i-1, r-1, c), G=(i-1, r-1, c-1), and H is a neighbor in the same slice but different row/column configuration as per Figure 2 in the source [S3].

### Angle Criterion for Neighbor Filtering
Filters the neighborhood $N(c)$ to remove neighbors that would produce inaccurate isopoints due to large angles between edges, ensuring only relevant neighbors are used for interpolation [S4]. The step involves sorting all points in $N(c)$ by distance to $x$, and deleting neighbors where the angle $\beta$ between the edge to that neighbor and the edge to any closer neighbor is less than a threshold $\alpha$ (typically $\alpha = \cos^{-1}(1/\sqrt{3})$) [S4]. This criterion is motivated by the desire to include diagonal neighbors in gridded data while maintaining consistency with three-dimensional optimal sphere packing [S4].

### Cluster Compactness Metric
A metric used to evaluate the quality of clusters identified by neighbor counting or clustering algorithms. The step involves initializing empty dictionaries and counters, identifying a valid mask where labels are $\ge 0$, and iterating over unique cluster IDs. For each cluster, it extracts points, calculates the centroid as the mean of the points, and computes the Mean Squared Error (MSE) as the mean of the sum of squared differences between the points and the centroid. The average compactness is calculated as the total weighted MSE divided by the total number of points [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** Building the KDTree takes $O(N \log N)$ on average. Querying neighbors for all points takes $O(N \log N)$ on average, but can degrade to $O(N^2)$ in worst-case scenarios (e.g., high-dimensional data or specific point distributions) [S1, unofficial].
*   **Memory Complexity:** $O(N)$ to store the points and the KDTree structure, plus $O(N)$ for the output array [S1, unofficial].
*   **Measured Performance:** Performance depends heavily on the dimensionality of the data and the distribution of points. For 3D point clouds, KDTree-based neighbor counting is generally efficient for $N < 10^6$ points. Not given in the sources for specific hardware benchmarks.
*   **Strengths:** Simple to implement, provides a direct measure of local density, efficient for static point clouds.
*   **Limitations:** Sensitive to the choice of $\epsilon$, computationally expensive for very large datasets or high dimensions, does not account for directionality of neighbors.
*   **When to use:** When a simple, fast estimate of local point density is needed for preprocessing, clustering, or noise filtering.
*   **When not to use:** When directional information is critical, or when the dataset is too large for in-memory KDTree construction.

## Background
Neighbor Counting is a core component of density-based spatial clustering algorithms like DBSCAN and is widely used in point cloud processing and volume rendering for density estimation [S1, unofficial]. The use of KDTree for efficient neighbor queries was popularized in the 1970s and remains a standard technique in computational geometry [S1, unofficial]. The specific application to volume rendering and point cloud density evaluation is discussed in recent works on dynamic geometry rendering and point-based surface representations [S1, unofficial][S4].

## Sources
[S1] wskingdom/DGR, https://github.com/wskingdom/DGR, fetched 2026-10-26 (unofficial)
[S2] Multi-Dimensional Transfer Function Design Using Sorted Histograms, https://doi.org/10.2312/vg/vg06/001-008, fetched 2026-10-26 (unofficial)
[S3] Employing New Hybrid Adaptive Wavelet-Based Transform and Histogram Packing to Improve JP3D Compression of Volumetric Medical Images, https://doi.org/10.3390/e22121385, fetched 2026-10-26
[S4] Direct Isosurface Extraction from Scattered Volume Data, https://doi.org/10.2312/vissym/eurovis06/099-106, fetched 2026-10-26
[S5] cilix-ai/gs-embedding, https://github.com/cilix-ai/gs-embedding, fetched 2026-10-27 (unofficial)
