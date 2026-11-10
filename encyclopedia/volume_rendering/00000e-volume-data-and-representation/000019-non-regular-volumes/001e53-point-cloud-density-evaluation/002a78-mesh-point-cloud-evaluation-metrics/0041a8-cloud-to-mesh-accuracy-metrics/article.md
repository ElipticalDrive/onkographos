# Cloud-to-Mesh Accuracy Metrics
Also called: Accuracy evaluation, Precision metrics
## Overview
Cloud-to-mesh accuracy metrics quantify how closely a reconstructed point cloud reflects ground truth locations by computing the displacements between each point in the compared point cloud and the nearest facet in the reference mesh [S1]. It serves as a precision measure, distinguishing itself from completeness (recall) metrics by focusing on the correctness of the reconstructed geometry rather than its coverage [S1]. It is a component of the broader [Mesh/Point Cloud Evaluation Metrics](4797-mesh-point-cloud-evaluation-metrics.md) family.
## How it works
**Inputs:**
1.  A reconstructed point cloud $P = \{p_1, p_2, \dots, p_n\}$ with $n$ points.
2.  A reference mesh $M$ consisting of a set of triangles (facets).

**Outputs:**
1.  Mean Error ($\bar{x}$).
2.  Standard Deviation ($SD$).
3.  Root Mean Square Error ($RMSE$).

**Data Structures:**
*   Point cloud: Array of $n$ 3D coordinates $(x, y, z)$.
*   Reference mesh: List of triangles, where each triangle is defined by 3 vertices.

**Steps:**
1.  For each point $p_i$ in the point cloud $P$:
    a.  Compute the Euclidean distance to the nearest facet in the reference mesh $M$.
    b.  The distance is the orthogonal (signed) distance from the point to the nearest triangle plane if the orthogonal projection of the point falls within the triangle bounds.
    c.  If the projection falls outside the triangle, the distance is to the nearest edge or vertex of that triangle [S1].
    d.  Store this minimum distance as $x_i$.
2.  Calculate the mean distance $\bar{x}$:
    $$ \bar{x} = \frac{\sum_{i=1}^{n} x_i}{n} $$
3.  Calculate the standard deviation $SD$:
    $$ SD = \sqrt{\frac{\sum_{i=1}^{n} (x_i - \bar{x})^2}{n-1}} $$
4.  Calculate the Root Mean Square Error $RMSE$:
    $$ RMSE = \sqrt{\frac{\sum_{i=1}^{n} (x_i - \bar{x})^2}{n}} $$

**Parameters:**
*   $n$: Number of points in the compared point cloud.
*   $x_i$: The closest distance between point $i$ and the nearest facet of the reference mesh [S1].

**Edge Cases and Pitfalls:**
*   The metric assumes a normal distribution without long-tailed data for the validity of Mean and SD as primary descriptors [S1].
*   Precision in determining whether a point's projection falls inside a triangle is critical; incorrect handling leads to inaccurate distance calculations [S1].
*   The metric is sensitive to outliers; a few points far from the mesh can significantly skew the Mean and RMSE [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_cloud_to_mesh_accuracy(point_cloud, reference_mesh):
    """
    Computes accuracy metrics for a point cloud against a reference mesh.
    
    Args:
        point_cloud: List of 3D points (n, 3).
        reference_mesh: List of triangles, each triangle is a list of 3 vertices (3D points).
        
    Returns:
        mean_error: Mean distance.
        std_dev: Standard deviation of distances.
        rmse: Root Mean Square Error.
    """
    n = len(point_cloud)
    distances = []
    
    for point in point_cloud:
        min_dist = infinity
        for triangle in reference_mesh:
            # Calculate distance from point to triangle
            dist = distance_point_to_triangle(point, triangle)
            if dist < min_dist:
                min_dist = dist
        distances.append(min_dist)
        
    distances = array(distances)
    
    # Mean Error
    mean_error = sum(distances) / n
    
    # Standard Deviation (sample standard deviation)
    variance = sum((d - mean_error)**2 for d in distances) / (n - 1)
    std_dev = sqrt(variance)
    
    # Root Mean Square Error
    rmse = sqrt(sum((d - mean_error)**2 for d in distances) / n)
    
    return mean_error, std_dev, rmse

def distance_point_to_triangle(point, triangle):
    """
    Computes the Euclidean distance from a point to a triangle.
    Returns the orthogonal distance if projection is inside, else distance to nearest edge/vertex.
    """
    # Implementation details for point-triangle distance are standard computational geometry
    # [G] This helper function implements the standard algorithm for point-to-triangle distance.
    # It checks barycentric coordinates to determine if the projection lies within the triangle.
    # If not, it computes distances to the three edges and three vertices, returning the minimum.
    pass
```
## Subtopics
## Operating characteristics
*   **Time Complexity:** $O(n \cdot t)$, where $n$ is the number of points in the cloud and $t$ is the number of triangles in the reference mesh. This is because each point must be compared against all triangles to find the nearest facet.
*   **Memory Complexity:** $O(n + t)$ to store the point cloud and mesh.
*   **Strengths:** Provides a direct measure of geometric precision; easy to interpret as an average displacement error [S1].
*   **Limitations:** Computationally expensive for large meshes due to the brute-force search; sensitive to outliers; assumes normal distribution for statistical validity [S1].
*   **When to use:** When evaluating the precision of reconstructed geometry against a high-fidelity ground truth mesh, particularly in scenarios where coverage (completeness) is secondary to accuracy [S1].
*   **When not to use:** When the mesh is extremely large and computational efficiency is critical; when the error distribution is not normal or has long tails [S1].
## Background
The metric was introduced in the context of evaluating 3D geometric reconstruction from Neural Radiance Fields (NeRFs) and Multi-View Stereo (MVS) [S1]. It is part of a broader evaluation framework that includes completeness metrics to assess both precision and recall of reconstructed point clouds [S1]. The method relies on standard statistical measures (Mean, SD, RMSE) applied to the distances between points and mesh facets [S1].
## Sources
[S1] Vision through Obstacles—3D Geometric Reconstruction and Evaluation of Neural Radiance Fields (NeRFs), https://doi.org/10.3390/rs16071188, fetched 2026-10-30
