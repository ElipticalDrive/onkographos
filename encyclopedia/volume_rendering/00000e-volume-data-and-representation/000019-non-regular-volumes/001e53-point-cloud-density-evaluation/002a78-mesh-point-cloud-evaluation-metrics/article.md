# Mesh/Point Cloud Evaluation Metrics


## Overview
This method computes a comprehensive suite of geometric fidelity metrics to evaluate the quality of reconstructed 3D meshes or point clouds against ground truth data. It addresses the problem of quantifying both surface proximity and geometric regularity, providing scalar scores for Chamfer Distance (CD), Normal Consistency (NC), F1 score, Edge Chamfer Distance (ECD), and mesh shape quality indicators [S1, unofficial]. It differs from standard mesh evaluation by explicitly calculating edge-specific metrics (ECD/EF1) and shape regularity (aspect ratio, angle regularity) alongside standard distance-based metrics, offering a more granular assessment of mesh topology and triangle quality [S1, unofficial].

## How it works
**Inputs:**
- Ground truth mesh or point cloud ($P_{gt}$).
- Predicted/Reconstructed mesh or point cloud ($P_{pred}$).
- Optional: Normals for both sets ($N_{gt}, N_{pred}$). If not provided, they must be estimated (e.g., via nearest neighbors).
- Parameters: `num_points` (default 100,000), `f1_threshold` (default 0.001), `ef1_radius` (default 0.004), `ef1_dotproduct_threshold` (default 0.2).

**Outputs:**
- Dictionary of metrics: CD1, CD2, NC, F1, ECD, EF1, Aspect Ratio, Radius Ratio, Angle Regularity, Self-Intersection %.

**Data Structures:**
- KDTree: Spatial index for $P_{gt}$ and $P_{pred}$ to enable efficient nearest neighbor queries.
- Point Arrays: $N \times 3$ float arrays for coordinates.
- Normal Arrays: $N \times 3$ float arrays for unit normals.

**Steps:**

1.  **Sampling:** If inputs are dense meshes, sample `num_points` uniformly from the surface of both $P_{gt}$ and $P_{pred}$ to create point sets $S_{gt}$ and $S_{pred}$. [S1, unofficial]
2.  **KDTree Construction:** Build a KDTree for $S_{gt}$ and another for $S_{pred}$. [S1, unofficial]
3.  **Chamfer Distance (CD):**
    -   For each point in $S_{pred}$, find the nearest neighbor in $S_{gt}$ (distance $d_{pred \to gt}$).
    -   For each point in $S_{gt}$, find the nearest neighbor in $S_{pred}$ (distance $d_{gt \to pred}$).
    -   Compute $CD_1 = \frac{1}{|S_{pred}|} \sum d_{pred \to gt} + \frac{1}{|S_{gt}|} \sum d_{gt \to pred}$ (L1).
    -   Compute $CD_2 = \sqrt{\frac{1}{|S_{pred}|} \sum d_{pred \to gt}^2 + \frac{1}{|S_{gt}|} \sum d_{gt \to pred}^2}$ (L2). [S1, unofficial]
4.  **Normal Consistency (NC):**
    -   For each point in $S_{pred}$, retrieve its nearest neighbor in $S_{gt}$ and the corresponding normals $n_{pred}$ and $n_{gt}$.
    -   Compute the dot product: $dot = n_{pred} \cdot n_{gt}$.
    -   Normalize if normals are not unit length: $dot_{norm} = \frac{dot}{\|n_{pred}\| \|n_{gt}\|}$. [S1, unofficial]
    -   $NC = \frac{1}{|S_{pred}|} \sum dot_{norm}$. Values range from -1 to 1, where 1 indicates perfect alignment. [S1, unofficial]
5.  **F1 Score:**
    -   Calculate Recall: $R = \frac{\text{count}(d_{pred \to gt} < \text{f1\_threshold})}{|S_{pred}|}$.
    -   Calculate Precision: $P = \frac{\text{count}(d_{gt \to pred} < \text{f1\_threshold})}{|S_{gt}|}$.
    -   $F1 = \frac{2 \cdot R \cdot P}{R + P}$. [S1, unofficial]
6.  **Edge Detection:**
    -   Identify "edge points" in $S_{pred}$ and $S_{gt}$ using a radius query. A point is an edge point if its nearest neighbor in the opposite set is within `ef1_radius` AND the dot product of their normals is less than `ef1_dotproduct_threshold` (indicating a sharp feature or boundary). [S1, unofficial]
7.  **Edge Chamfer Distance (ECD) and Edge F1 (EF1):**
    -   Compute CD only on the subset of edge points identified in Step 6.
    -   Compute F1 only on the subset of edge points using `ef1_threshold` (default 0.005). [S1, unofficial]
8.  **Mesh Shape Quality (if inputs are meshes):**
    -   **Aspect Ratio:** For each triangle $t$ in the mesh, compute the inertia matrix $M_t$. Find eigenvalues $\lambda_1 \ge \lambda_2$. $\rho(t) = \lambda_2 / \lambda_1$. Average over all triangles. $\rho(t)=1$ for equilateral, $0$ for degenerate. [S2]
    -   **Radius Ratio:** Ratio of circumradius to inradius of triangles. [S1, unofficial]
    -   **Angle Regularity:** Measure of deviation from equiangular triangles. [S1, unofficial]
9.  **Self-Intersection:**
    -   Check for self-intersecting triangles in the predicted mesh. Report percentage of intersecting triangles. [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.spatial import KDTree

def compute_mesh_point_cloud_metrics(gt_points, gt_normals, pred_points, pred_normals, 
                                     f1_threshold=0.001, ef1_radius=0.004, 
                                     ef1_dot_threshold=0.2, num_points=100000):
    """
    Computes CD, NC, F1, ECD, EF1, and mesh quality metrics.
    Assumes gt_points/pred_points are already sampled or are the point clouds.
    If meshes, sampling logic must be applied before this function.
    """
    
    # 1. Build KDTrees
    tree_gt = KDTree(gt_points)
    tree_pred = KDTree(pred_points)
    
    # 2. Nearest Neighbor Queries
    # pred -> gt
    dist_pred_to_gt, idx_pred_to_gt = tree_gt.query(pred_points, k=1)
    # gt -> pred
    dist_gt_to_pred, idx_gt_to_pred = tree_pred.query(gt_points, k=1)
    
    # 3. Chamfer Distance
    cd1 = np.mean(dist_pred_to_gt) + np.mean(dist_gt_to_pred)
    cd2 = np.sqrt(np.mean(dist_pred_to_gt**2) + np.mean(dist_gt_to_pred**2))
    
    # 4. Normal Consistency
    # Get normals for the matched points
    pred_normals_matched = pred_normals[idx_gt_to_pred] # For each gt point, get pred normal
    gt_normals_matched = gt_normals[idx_pred_to_gt]     # For each pred point, get gt normal
    
    # Dot products
    # Note: Ensure normals are normalized
    dot_products_pred = np.sum(pred_normals * gt_normals_matched, axis=1) / (np.linalg.norm(pred_normals, axis=1) * np.linalg.norm(gt_normals_matched, axis=1))
    dot_products_gt = np.sum(gt_normals * pred_normals_matched, axis=1) / (np.linalg.norm(gt_normals, axis=1) * np.linalg.norm(pred_normals_matched, axis=1))
    
    # NC is typically computed as the mean dot product of corresponding normals
    # Here we average the consistency in both directions or just one? 
    # Standard practice often uses the mean of dot products of corresponding pairs.
    # We align them by index. Let's assume we evaluate NC on the pred points against their gt matches.
    nc = np.mean(dot_products_pred) 
    
    # 5. F1 Score
    recall = np.sum(dist_pred_to_gt < f1_threshold) / len(pred_points)
    precision = np.sum(dist_gt_to_pred < f1_threshold) / len(gt_points)
    f1 = 2 * recall * precision / (recall + precision) if (recall + precision) > 0 else 0.0
    
    # 6. Edge Detection
    # Edge points are those with close neighbors AND low normal consistency (sharp features)
    # We define edge points in pred based on their relationship to gt
    is_edge_pred = (dist_pred_to_gt < ef1_radius) & (dot_products_pred < ef1_dot_threshold)
    is_edge_gt = (dist_gt_to_pred < ef1_radius) & (dot_products_gt < ef1_dot_threshold)
    
    edge_pred_points = pred_points[is_edge_pred]
    edge_pred_normals = pred_normals[is_edge_pred]
    edge_gt_points = gt_points[is_edge_gt]
    edge_gt_normals = gt_normals[is_edge_gt]
    
    # 7. ECD and EF1
    if len(edge_pred_points) > 0 and len(edge_gt_points) > 0:
        tree_edge_gt = KDTree(edge_gt_points)
        tree_edge_pred = KDTree(edge_pred_points)
        
        dist_edge_pred_to_gt, _ = tree_edge_gt.query(edge_pred_points, k=1)
        dist_edge_gt_to_pred, _ = tree_edge_pred.query(edge_gt_points, k=1)
        
        ecd = np.mean(dist_edge_pred_to_gt) + np.mean(dist_edge_gt_to_pred)
        
        ef1_recall = np.sum(dist_edge_pred_to_gt < f1_threshold) / len(edge_pred_points)
        ef1_precision = np.sum(dist_edge_gt_to_pred < f1_threshold) / len(edge_gt_points)
        ef1 = 2 * ef1_recall * ef1_precision / (ef1_recall + ef1_precision) if (ef1_recall + ef1_precision) > 0 else 0.0
    else:
        ecd = 0.0
        ef1 = 0.0
        
    # 8. Mesh Quality Metrics (Requires Mesh Structure, not just points)
    # If inputs are point clouds, these metrics are not applicable or require mesh reconstruction first.
    # Assuming mesh inputs were passed or reconstructed:
    aspect_ratio = 0.0 # Placeholder: Computed via inertia matrix eigenvalues of triangles
    radius_ratio = 0.0 # Placeholder
    angle_regularity = 0.0 # Placeholder
    self_intersection_pct = 0.0 # Placeholder
    
    return {
        "CD1": cd1,
        "CD2": cd2,
        "NC": nc,
        "F1": f1,
        "ECD": ecd,
        "EF1": ef1,
        "Aspect_Ratio": aspect_ratio,
        "Radius_Ratio": radius_ratio,
        "Angle_Regularity": angle_regularity,
        "Self_Intersection_Pct": self_intersection_pct
    }
```

**Edge Cases and Pitfalls:**
-   **Precision:** Chamfer distance is sensitive to outliers. Use L2 for robustness if outliers are expected, but L1 is more common for surface proximity [S1, unofficial].
-   **Boundaries:** Edge metrics (ECD/EF1) are critical for capturing boundary fidelity, which global CD might miss. Ensure `ef1_radius` is scaled to the object size [S1, unofficial].
-   **Degenerate Input:** If the predicted mesh has no triangles or points, metrics should return 0 or NaN with a warning. Aspect ratio is 0 for degenerate triangles [S2].
-   **Sampling Bias:** Uniform sampling may undersample small features. Stratified sampling or importance sampling based on curvature is recommended but not specified in the base method [S1, unofficial].

## Subtopics

### [Evaluation Pipeline](9358-evaluation-pipeline.md)
This pipeline computes Chamfer distance ($C_{\ell1}$), intersection-over-union (IoU), normal consistency (NC), and F-score on reconstructed surfaces that are visible in the input color/depth streams. It differs from standard mesh evaluation by explicitly culling occluded or unseen geometry to prevent error inflation from regions where the network output is unpredictable or where ground truth contains hidden surfaces. It builds on [Mesh/Point Cloud Evaluation Metrics](4797-mesh-point-cloud-evaluation-metrics.md) but restricts the comparison domain to the viewing frusta of the input cameras.

### [Evaluation Metrics](9899-evaluation-metrics.md)
Computes quantitative assessment of rendering quality against ground truth using Peak Signal-to-Noise Ratio (PSNR) and Structural Similarity Index Measure (SSIM). Additionally, it assesses mesh quality using Chamfer Distance (CD), F1 score, and Normal Consistency (NC) to evaluate geometric fidelity. It differs from the [Evaluation Pipeline](9358-evaluation-pipeline.md) by focusing specifically on the mathematical computation of these scalar metrics rather than the orchestration of data loading and training loops.

### Mesh Quality Evaluation
A metric to evaluate triangle shape quality, preferring well-shaped triangles over long and skinny ones. The aspect ratio of triangle $t=abc$ is $\rho(t) = \lambda_2 / \lambda_1$, where $\lambda_1 \ge \lambda_2$ are the greatest eigenvalues of the inertia matrix of $t$. $\rho(t) = 1$ if the triangle is equilateral ($\lambda_1 = \lambda_2$) and $\rho(t) = 0$ if it is degenerate ($\lambda_2 = 0$) [S2].

### Cloud-to-Mesh Accuracy Metrics
Quantifies how closely the reconstructed point cloud reflects ground truth locations by computing displacements to the nearest facet. Mean Error (Mean) is $Mean = \frac{\sum_{i=1}^{n} (x_i)}{n}$, where $n$ is the number of points and $x_i$ is the closest distance between point $i$ and the nearest facet. Standard Deviation (SD) is $SD = \sqrt{\frac{\sum_{i=1}^{n} (x_i - \bar{x})^2}{n-1}}$, where $\bar{x}$ is the mean value of the distances [S3].

## Operating characteristics
-   **Time Complexity:** Dominated by KDTree construction $O(N \log N)$ and $K$-nearest neighbor queries $O(N \log N)$, where $N$ is the number of points. Edge detection adds a constant factor overhead.
-   **Memory Complexity:** $O(N)$ to store points, normals, and KDTree structures.
-   **Performance:** Efficient for point clouds up to millions of points on modern hardware. Mesh quality metrics (aspect ratio, self-intersection) require additional geometric processing and may be slower if the mesh is highly subdivided.
-   **Strengths:** Provides a holistic view of geometric quality, including boundary fidelity (ECD/EF1) and shape regularity. Robust to minor surface noise due to distance thresholds.
-   **Limitations:** Does not account for occlusion unless explicitly integrated (see [Evaluation Pipeline](9358-evaluation-pipeline.md)). Sensitive to sampling density; mismatched sampling between GT and Pred can skew CD.
-   **When to use:** When evaluating 3D reconstruction algorithms (NeRF, Gaussian Splatting, Mesh extraction) where geometric fidelity is as important as visual fidelity.
-   **When not to use:** For purely visual quality assessment (use PSNR/SSIM) or when occlusion handling is critical without explicit culling logic.

## Background
The method originates from the evaluation protocols used in 3D deep learning frameworks, specifically those involving neural implicit representations and mesh extraction. The specific combination of CD, NC, F1, and edge metrics is detailed in the TetWeave repository [S1, unofficial]. Aspect ratio and mesh quality metrics are derived from classical computational geometry literature on isosurface extraction [S2]. Cloud-to-mesh accuracy metrics (Mean, SD) are standard in remote sensing and photogrammetry evaluation [S3].

## Sources
[S1] AlexandreBinninger/TetWeave, https://github.com/AlexandreBinninger/TetWeave, fetched 2026-10-27 (unofficial)
[S2] Isosurface extraction and interpretation on very large datasets in geophysics, https://inria.hal.science/inria-00277070v1, fetched 2026-10-26
[S3] Vision through Obstacles—3D Geometric Reconstruction and Evaluation of Neural Radiance Fields (NeRFs), https://doi.org/10.3390/rs16071188, fetched 2026-10-30
