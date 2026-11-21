# Cloud-to-Mesh Accuracy Metrics

Also called: Accuracy evaluation, Precision metrics

## Overview
Cloud-to-Mesh Accuracy Metrics quantify how closely a reconstructed point cloud reflects ground truth locations by computing the Euclidean displacements from each point in the compared cloud to the nearest facet of a reference mesh [S1]. Unlike symmetric metrics that average errors in both directions, this method specifically evaluates the precision of the reconstruction by measuring the orthogonal distance to the reference surface, handling cases where the projection falls outside the triangle boundaries by measuring to the nearest edge [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A reconstructed point cloud $P = \{p_1, p_2, \dots, p_n\}$ with $n$ points, and a reference mesh $M$ consisting of triangles [S1].
*   **Output:** Statistical measures of geometric accuracy: Mean Error, Standard Deviation (SD), and Root Mean Square Error (RMSE) [S1].

**Data Structures:**
*   The reference mesh $M$ is represented as a collection of triangles, where each triangle is defined by three vertices and associated with a normal vector [G].
*   The point cloud $P$ is a list of 3D coordinates [G].

**Steps:**
1. For each point $p_i$ in the reconstructed cloud $P$:
    a. Find the nearest facet (triangle) in the reference mesh $M$ [S1].
    b. Calculate the orthogonal (signed) distance from $p_i$ to the plane of the nearest triangle [S1].
    c. Check if the orthogonal projection of $p_i$ falls within the boundaries of the triangle [S1].
    d. If the projection falls outside the triangle, calculate the distance to the nearest edge of that triangle [S1].
    e. Let $x_i$ be the final calculated distance (minimum of the planar distance or edge distance) [S1].
2. Compute the statistics based on the set of distances $\{x_1, x_2, \dots, x_n\}$ [S1].

**Formulas:**
Let $n$ be the number of points in the compared point cloud, and $x_i$ be the closest distance between point $i$ and the nearest facet of the reference mesh [S1].

The Mean Error ($\bar{x}$) is:
$$ \bar{x} = \frac{\sum_{i=1}^{n} x_i}{n} $$

The Standard Deviation (SD) is:
$$ SD = \sqrt{\frac{\sum_{i=1}^{n} (x_i - \bar{x})^2}{n-1}} $$

The Root Mean Square Error (RMSE) is:
$$ RMSE = \sqrt{\frac{\sum_{i=1}^{n} (x_i - \bar{x})^2}{n}} $$

**Parameters:**
*   $n$: Number of points in the reconstructed cloud [S1].
*   Distance threshold: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Projection Outside Triangle:** The method explicitly handles points whose orthogonal projection does not fall on any facet by measuring the distance to the nearest edge [S1].
*   **Data Distribution:** The metrics assume a normal distribution without long-tailed data for the validity of Mean and SD as primary descriptors [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_cloud_to_mesh_accuracy(points, mesh):
    """
    Computes Mean, SD, and RMSE for a point cloud against a reference mesh.
    
    Args:
        points: List of 3D coordinates (n, 3).
        mesh: Object containing triangles (list of vertex triplets).
        
    Returns:
        dict: Mean, SD, RMSE.
    """
    distances = []
    
    for p in points:
        min_dist = float('inf')
        for tri in mesh.triangles:
            v0, v1, v2 = tri
            # Point-to-triangle distance calculation [G]
            e1 = [v1[k] - v0[k] for k in range(3)]
            e2 = [v2[k] - v0[k] for k in range(3)]
            d = [p[k] - v0[k] for k in range(3)]
            a = sum(e1[k]**2 for k in range(3))
            b = sum(e1[k]*e2[k] for k in range(3))
            c = sum(e2[k]**2 for k in range(3))
            d1 = sum(e1[k]*d[k] for k in range(3))
            d2 = sum(e2[k]*d[k] for k in range(3))
            denom = a*c - b*b
            if denom == 0:
                u, v = 0.0, 0.0
            else:
                u = (b*c - d1*c) / denom
                v = (b*d1 - a*d2) / denom
            if u < 0.0:
                u = 0.0
                v = max(0.0, (b*d2 - c*d1) / (b*b - a*c)) if b*b != a*c else 0.0
            if v < 0.0:
                v = 0.0
                u = max(0.0, (b*d1 - a*d2) / (b*b - a*c)) if b*b != a*c else 0.0
            if u + v > 1.0:
                if u > 0.5:
                    u, v = 1.0, 0.0
                else:
                    u, v = 0.0, 1.0
            closest = [v0[k] + u*e1[k] + v*e2[k] for k in range(3)]
            dist = math.sqrt(sum((p[k] - closest[k])**2 for k in range(3)))
            if dist < min_dist:
                min_dist = dist
        distances.append(min_dist)
        
    n = len(distances)
    if n == 0:
        return {"Mean": 0.0, "SD": 0.0, "RMSE": 0.0}
        
    mean_val = sum(distances) / n
    variance = sum((x - mean_val)**2 for x in distances) / (n - 1)
    sd_val = math.sqrt(variance)
    rmse_val = math.sqrt(sum((x - mean_val)**2 for x in distances) / n)
    
    return {"Mean": mean_val, "SD": sd_val, "RMSE": rmse_val}
```

## Subtopics

### [L1-absolute Error (Accuracy)](12111-l1-absolute-error-accuracy.md)
L1-absolute Error quantifies the geometric accuracy of a reconstructed 3D model by measuring the average signed distance field (SDF) deviation of points sampled from a high-fidelity reference mesh against coarser or estimated reconstructions. It is specifically designed for evaluating TSDF (Truncated Signed Distance Field) or SDF-based volume representations, distinguishing itself from point-to-mesh Euclidean distance metrics by operating directly in the implicit field space.

### [ROC Geometry Matching Evaluation](14003-roc-geometry-matching-evaluation.md)
ROC Geometry Matching Evaluation quantifies the geometric accuracy of a reconstructed point cloud $B$ against a reference point cloud $A$ by computing the True Positive Rate (TPR) and False Positive Rate (FPR) at a specific distance threshold. Unlike symmetric metrics such as D1/D2 that average errors in both directions and assume similar point counts, this asymmetric method handles cases where the predicted cloud $B$ contains significantly more points than the reference $A$, preventing artificially inflated accuracy scores in the $A \rightarrow B$ direction.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Provides a direct measure of precision (how close the points are to the truth) [S1].
*   **Limitations:** It is asymmetric; it does not measure completeness (recall) [S1]. A cloud with few points but all correctly placed will have high accuracy but low completeness [S1]. It assumes the reference mesh is a valid surface representation [G].
*   **When to use:** When the primary concern is the geometric fidelity of the reconstructed points to the ground truth surface [S1].

## Background
The Cloud-to-Mesh Accuracy Metric was introduced in the context of evaluating 3D geometric reconstruction from Neural Radiance Fields (NeRFs) and Multi-View Stereo (MVS) [S1]. It is part of a broader evaluation framework that includes completeness metrics, aiming to provide a comprehensive assessment of reconstruction quality by considering both precision and recall [S1]. The method relies on standard statistical measures (Mean, SD, RMSE) applied to the distribution of distances between the reconstructed points and the reference mesh [S1].

## Sources
[S1] Vision through Obstacles—3D Geometric Reconstruction and Evaluation of Neural Radiance Fields (NeRFs), https://doi.org/10.3390/rs16071188, fetched 2026-10-30
