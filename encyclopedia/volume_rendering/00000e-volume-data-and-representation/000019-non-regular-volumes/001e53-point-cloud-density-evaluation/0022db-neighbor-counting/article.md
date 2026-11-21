# Neighbor Counting


## Overview
Neighbor Counting is a fundamental technique in volume data and representation that quantifies the local density or spatial distribution of points within a point cloud or mesh by determining how many other points reside within a specified proximity of a query point [S1, unofficial]. It serves as the primary mechanism for evaluating point cloud density in non-regular volumes, distinguishing itself from continuous density estimators like [Parzen Density Estimation](11796-parzen-density-estimation.md) by providing discrete, integer-based counts rather than smoothed kernel sums [S1, unofficial]. Unlike connectivity-based clustering methods such as [Grid-based DBSCAN](11706-grid-based-dbscan.md) which group points into arbitrary shapes, Neighbor Counting focuses on local neighborhood statistics to inform voxelization, surface reconstruction, or clustering decisions [S1, unofficial].

## How it works
The method operates on a set of input points $P = \{p_1, p_2, \dots, p_n\}$ in $\mathbb{R}^3$ and a distance threshold $\tau$. The output is a set of neighbor counts $C = \{c_1, c_2, \dots, c_n\}$, where $c_i$ represents the number of neighbors of point $p_i$ within distance $\tau$.

### Data Structures
1.  **Input Points**: A list or array of 3D coordinates $(x, y, z)$.
2.  **KDTree**: A spatial index structure built from the input points to enable efficient range queries. The tree nodes store point indices and bounding boxes.
3.  **Neighbor Counts**: An integer array of size $n$, initialized to zero.

### Parameters
*   $\tau$ (distance_threshold): The maximum Euclidean distance to consider a point a neighbor. Typical values depend on the scale of the input data; for normalized unit cubes, values like $0.1$ to $0.5$ are common. Not given in the sources for specific standard values, but implied by the query radius usage [S1, unofficial].

### Steps
1.  **Build Index**: Construct a KDTree from the input point set $P$.
2.  **Query**: For each point $p_i \in P$:
    *   Query the KDTree for all points within Euclidean distance $\tau$ of $p_i$.
    *   Let $N_i$ be the set of returned points.
    *   Calculate the count $c_i = |N_i| - 1$. The subtraction of 1 excludes the point $p_i$ itself, which is always within distance 0.
3.  **Store**: Save $c_i$ to the output array at index $i$.

### Edge Cases and Pitfalls
*   **Self-Inclusion**: The query radius typically includes the query point itself. The algorithm must explicitly subtract 1 from the raw count to reflect *other* neighbors [S1, unofficial].
*   **Boundary Conditions**: Points near the boundary of the point cloud may have artificially low counts due to the absence of points outside the dataset. This is an inherent property of the method and not an error, but must be considered when interpreting density [S1, unofficial].
*   **Precision**: Floating-point precision in distance calculations can affect which points are included if they lie exactly on the boundary $\tau$. Standard Euclidean distance comparison `dist <= tau` is recommended.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
from typing import List, Tuple

def euclidean_distance(p1: Tuple[float, float, float], p2: Tuple[float, float, float]) -> float:
    """Calculate Euclidean distance between two 3D points."""
    return math.sqrt(
        (p1[0] - p2[0])**2 +
        (p1[1] - p2[1])**2 +
        (p1[2] - p2[2])**2
    )

class KDTree:
    """Simplified KDTree for range queries. In practice, use a library or optimized implementation."""
    def __init__(self, points: List[Tuple[float, float, float]]):
        self.points = points
        self.n = len(points)
        # Note: A real implementation would build a tree structure.
        # For the purpose of this article, we assume a query_radius method exists
        # that returns indices of points within distance r.
    
    def query_radius(self, point: Tuple[float, float, float], radius: float) -> List[int]:
        """Return indices of all points within 'radius' of 'point'."""
        # Placeholder for actual KDTree logic. 
        # In a real implementation, this is O(log n) on average.
        # Here we simulate the result for clarity of the algorithm step.
        neighbors = []
        for i, p in enumerate(self.points):
            if euclidean_distance(point, p) <= radius:
                neighbors.append(i)
        return neighbors

def neighbor_counting(points: List[Tuple[float, float, float]], distance_threshold: float) -> List[int]:
    """
    Calculate the number of neighbors for each point within the given distance threshold.
    
    Args:
        points: List of 3D coordinates.
        distance_threshold: Maximum distance to consider a neighbor.
        
    Returns:
        List of neighbor counts (excluding self).
    """
    if not points:
        return []
    
    # Step 1: Build KDTree
    tree = KDTree(points)
    n = len(points)
    counts = [0] * n
    
    # Step 2: Query for each point
    for i, p in enumerate(points):
        # Query returns indices including self
        neighbor_indices = tree.query_radius(p, distance_threshold)
        
        # Step 3: Subtract 1 to exclude self
        # The count is the number of *other* points
        count = len(neighbor_indices) - 1
        
        # Handle case where no other points are within range (count becomes 0)
        if count < 0:
            count = 0
            
        counts[i] = count
        
    return counts
```

## Subtopics

### [Clusterability Check](5598-clusterability-check.md)
Determines if a K-D tree node's grid can be clustered by verifying if the combined QEF point count matches the sum of component point counts. It acts as a binary gate in the Neighbor Counting pipeline, preventing invalid aggregation of geometric primitives.

### [KDTree-based Neighbor Counting](6285-kdtree-based-neighbor-counting.md)
Calculates density by querying neighboring points using a KDTree and normalizing counts. It uses a fixed spherical radius, differing from [Grid-based DBSCAN](11706-grid-based-dbscan.md) which uses connectivity, and from [Parzen Density Estimation](11796-parzen-density-estimation.md) which provides smoothed sums.

### [Grid-based DBSCAN](11706-grid-based-dbscan.md)
An optimized 2D DBSCAN variant that maps data to a grid and uses graph connectivity. It guarantees $O(n \log n)$ complexity for 2D data, unlike tree-based methods which may not guarantee this consistency.

### [DBSCAN](11722-dbscan.md)
A density-based clustering algorithm that identifies arbitrary-shaped clusters and handles noise. It distinguishes itself from centroid-based methods by detecting non-convex shapes and classifying outliers.

### [Parzen Density Estimation](11796-parzen-density-estimation.md)
Estimates the probability density function using continuous Gaussian kernel interpolation. It provides smoother decision boundaries than discrete neighbor counts, useful for connecting clusters across quantized slices.

### Range Weight
A statistical neighborhood analysis attribute derived from intensity values that describes the neighborhood footprint in a given partial intensity range $\Phi$, acting as a proxy for material characteristics like homogeneity and size [S2, unofficial]. It is defined as:
$$ w_r(\Phi, N) = \frac{|N \cap V_\Phi|}{|N|} $$
where $N$ is an arbitrary voxel neighborhood, $V_\Phi$ is the set of voxels within range $\Phi$, and $|V|$ is the cardinality of set $V$ [S2, unofficial]. It is used in forensic visualizations to highlight metal fragments and in MR bile duct examinations to isolate liver tumors [S2, unofficial].

### Predictor Definitions
In the context of adaptive wavelet-based transform construction for volumetric medical image compression, neighbor counts inform predictor selection. The layout defines predictors based on neighbors A, B, C, D, E, F, G, H relative to a target pixel X at $(i, r, c)$ [S3]:
*   **NOP**: 0
*   **P_X**: A (Left neighbor)
*   **P_Y**: B (Top neighbor)
*   **P_Z**: D (Previous slice neighbor)
*   **AVG_XY**: $(A+B)/2$
*   **AVG_XZ**: $(A+D)/2$
*   **AVG_YZ**: $(B+D)/2$
*   **MED_XY**: $\text{median}(A, B, A+B-C)$
*   **MED_XZ**: $\text{median}(A, D, A+D-E)$
*   **MED_YZ**: $\text{median}(B, D, B+D-F)$
*   **AVG3D**: $(A+B+D)/3$
Neighbor locations are defined spatially: A=(i, r, c-1), B=(i, r-1, c), C=(i, r-1, c-1), D=(i-1, r, c), E=(i-1, r, c-1), F=(i-1, r-1, c), G=(i-1, r-1, c-1), H=(i, r-1, c+1) [S3].

### Angle Criterion for Neighbor Filtering
To ensure accurate isopoint interpolation, the neighborhood $N(c)$ of a point $x$ is filtered by an angle criterion. All points in $N(c)$ are sorted by distance to $x$. Neighbors are deleted if the angle $\beta$ between the edge to that neighbor and the edge to any closer neighbor is less than a threshold $\alpha$ [S4]. A typical choice for $\alpha$ is $\cos^{-1}(1/\sqrt{3})$, consistent with 3D optimal sphere packing, ensuring inclusion of diagonal neighbors in gridded data [S4]. This filtering breaks symmetry, potentially requiring duplicate isopoint computations [S4].

### Cluster Compactness Metric
A metric used to evaluate the quality of clusters identified via neighbor-based methods. It calculates the Mean Squared Error (MSE) of points relative to their cluster centroid. The algorithm initializes `compactness` dict, `total_weighted`, and `total_points`. It iterates over unique cluster IDs, extracts points, calculates the centroid, computes MSE, and stores it. The average compactness is the weighted sum of MSEs divided by total points [S5, unofficial].

## Operating characteristics
*   **Time Complexity**: Building the KDTree takes $O(n \log n)$ on average. Querying all points takes $O(n \log n)$ on average for uniform distributions, but can degrade to $O(n^2)$ in worst-case scenarios (e.g., highly clustered data or poor tree balancing) [S1, unofficial].
*   **Memory Complexity**: $O(n)$ to store the points and the KDTree structure [S1, unofficial].
*   **Strengths**: Simple to implement, provides exact discrete counts, efficient with spatial indexing [S1, unofficial].
*   **Limitations**: Sensitive to the choice of distance threshold $\tau$; does not account for anisotropy or directionality; performance degrades in high dimensions without specialized indexing [S1, unofficial].
*   **When to Use**: When discrete density estimates are needed for voxelization, clustering initialization, or surface reconstruction filtering [S1, unofficial].
*   **When Not to Use**: When smooth density gradients are required (use [Parzen Density Estimation](11796-parzen-density-estimation.md)) or when arbitrary-shaped cluster detection is the primary goal (use [DBSCAN](11722-dbscan.md)) [S1, unofficial].

## Background
Neighbor Counting is a foundational technique in computational geometry and computer graphics, widely used in point cloud processing and volume rendering. Its origins are tied to the development of spatial indexing structures like KD-Trees in the 1970s. In the context of volume rendering, it is employed for density evaluation in non-regular volumes, such as point-based rendering and mesh simplification. The method is cited in recent works on direct isosurface extraction from scattered volume data and point cloud density evaluation [S1, unofficial][S4].

## Sources
[S1] wskingdom/DGR, https://github.com/wskingdom/DGR, fetched 2026-10-26 (unofficial)
[S2] Multi-Dimensional Transfer Function Design Using Sorted Histograms, https://doi.org/10.2312/vg/vg06/001-008, fetched 2026-10-26 (unofficial)
[S3] Employing New Hybrid Adaptive Wavelet-Based Transform and Histogram Packing to Improve JP3D Compression of Volumetric Medical Images, https://doi.org/10.3390/e22121385, fetched 2026-10-26
[S4] Direct Isosurface Extraction from Scattered Volume Data, https://doi.org/10.2312/vissym/eurovis06/099-106, fetched 2026-10-26
[S5] cilix-ai/gs-embedding, https://github.com/cilix-ai/gs-embedding, fetched 2026-10-27 (unofficial)
