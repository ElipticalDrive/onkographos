# Mesh/Point Cloud Evaluation Metrics


## Overview
Computes comprehensive mesh quality metrics including Chamfer Distance (CD), Normal Consistency (NC), F1 score, Edge Chamfer Distance (ECD), Edge F1 (EF1), aspect ratio, radius ratio, angle regularity, and self-intersection percentage [S1, unofficial]. It differs from standard Chamfer Distance by incorporating edge-specific metrics and geometric shape quality measures, and from Neighbor Counting by operating on explicit mesh topology or sampled point clouds with normal information rather than just voxel occupancy [S1, unofficial][S3].

## How it works
**Inputs:**
- Ground truth mesh/point cloud and predicted mesh/point cloud.
- Optional: Camera trajectory for visibility culling [S3].
- Parameters: `f1_threshold` (default 0.001), `ef1_threshold` (default 0.005), `ef1_radius` (default 0.004), `ef1_dotproduct_threshold` (default 0.2), `num_points` (default 100000) [S1, unofficial].

**Outputs:**
- Dictionary of metrics: CD1, CD2, NC, F1, ECD, EF1, aspect ratio, radius ratio, angle regularity, self-intersection percentage [S1, unofficial].

**Data Structures:**
- KDTree for efficient nearest neighbor queries on point sets [S1, unofficial].
- Point arrays with associated normals (if available) [S1, unofficial].

**Steps:**
1. **Sampling:** If inputs are meshes, sample `num_points` points. If inputs are point clouds, ensure they are sampled to a consistent density [S1, unofficial][S3]. For visibility-aware evaluation, subdivide meshes to ensure maximum edge length < 1.5 cm, detect visible vertices using ground truth camera trajectory, and cull triangles with zero visible vertices (occluded or outside viewing frusta) [S3]. Sample culled geometry at a density of 1 point per cm² [S3].
2. **KDTree Construction:** Build KDTree for both prediction and ground truth points [S1, unofficial].
3. **Chamfer Distance (CD):** Query nearest neighbors (k=1) for both directions (gt→pred, pred→gt). Compute distances for CD1 (L1) and CD2 (L2) [S1, unofficial].
4. **Normal Consistency (NC):** Compute dot product of normals between corresponding points. Normalize by magnitudes if normals are not unit length [S1, unofficial].
5. **F1 Score:** Compute recall and precision using `f1_threshold`.
   - Recall = $\frac{\sum (\text{dist} < \text{f1\_threshold})}{\text{len}(\text{dist})}$
   - Precision = $\frac{\sum (\text{dist} < \text{f1\_threshold})}{\text{len}(\text{dist})}$
   - F1 = $\frac{2 \times \text{recall} \times \text{precision}}{\text{recall} + \text{precision}}$ [S1, unofficial].
6. **Edge Detection:** Identify edge points using `query_radius` with `ef1_radius` and `ef1_dotproduct_threshold` [S1, unofficial].
7. **Edge Metrics:** Compute ECD and EF1 on the identified edge points using `ef1_threshold` [S1, unofficial].
8. **Geometric Quality:** Calculate aspect ratio and radius ratio via mesh analysis tools (e.g., pyvista) [S1, unofficial]. Compute angle regularity via mesh analysis tools (e.g., trimesh) [S1, unofficial].
9. **Self-Intersection:** Compute self-intersection percentage [S1, unofficial].
10. **Return:** Aggregated metrics [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.spatial import KDTree

def compute_mesh_point_cloud_metrics(gt_points, gt_normals, pred_points, pred_normals, 
                                     f1_threshold=0.001, ef1_threshold=0.005, 
                                     ef1_radius=0.004, ef1_dotproduct_threshold=0.2, 
                                     num_points=100000):
    """
    Compute comprehensive mesh quality metrics.
    
    Args:
        gt_points: (N, 3) array of ground truth points.
        gt_normals: (N, 3) array of ground truth normals.
        pred_points: (M, 3) array of predicted points.
        pred_normals: (M, 3) array of predicted normals.
        f1_threshold: Distance threshold for F1 score.
        ef1_threshold: Distance threshold for Edge F1.
        ef1_radius: Radius for edge point detection.
        ef1_dotproduct_threshold: Dot product threshold for edge classification.
        num_points: Number of points to sample if inputs are meshes (not used here as inputs are points).
        
    Returns:
        Dictionary of metrics.
    """
    # 1. Build KDTree
    gt_tree = KDTree(gt_points)
    pred_tree = KDTree(pred_points)
    
    # 2. Query nearest neighbors (k=1)
    # gt->pred
    dist_gt_to_pred, idx_gt_to_pred = gt_tree.query(pred_points, k=1)
    # pred->gt
    dist_pred_to_gt, idx_pred_to_gt = pred_tree.query(gt_points, k=1)
    
    # 3. Compute Chamfer Distance (CD)
    cd1 = np.mean(dist_gt_to_pred) + np.mean(dist_pred_to_gt)
    cd2 = np.sqrt(np.mean(dist_gt_to_pred**2) + np.mean(dist_pred_to_gt**2))
    
    # 4. Compute Normal Consistency (NC)
    # Assuming normals are already normalized. If not, normalize them.
    # dotproduct = (np.sum(gt_normals*neighbor_normals, axis=1)) / (np.linalg.norm(gt_normals, axis=1) * np.linalg.norm(neighbor_normals, axis=1))
    # Here, neighbor_normals for gt_points are pred_normals[idx_pred_to_gt]
    # And neighbor_normals for pred_points are gt_normals[idx_gt_to_pred]
    
    # For NC, we typically compare corresponding points. 
    # However, CD is bidirectional. NC is often computed as the mean cosine similarity 
    # between corresponding points in both directions or just one if aligned.
    # Let's compute mean cosine similarity for gt->pred and pred->gt pairs.
    
    # gt points matched to pred points
    pred_normals_for_gt = pred_normals[idx_gt_to_pred]
    nc_gt_to_pred = np.mean(np.sum(gt_normals * pred_normals_for_gt, axis=1))
    
    # pred points matched to gt points
    gt_normals_for_pred = gt_normals[idx_pred_to_gt]
    nc_pred_to_gt = np.mean(np.sum(pred_normals * gt_normals_for_pred, axis=1))
    
    nc = (nc_gt_to_pred + nc_pred_to_gt) / 2.0
    
    # 5. Compute F1 score
    # Recall: fraction of gt points within f1_threshold of pred points
    recall = np.mean(dist_pred_to_gt < f1_threshold)
    # Precision: fraction of pred points within f1_threshold of gt points
    precision = np.mean(dist_gt_to_pred < f1_threshold)
    
    if recall + precision == 0:
        f1 = 0.0
    else:
        f1 = 2 * recall * precision / (recall + precision)
        
    # 6. Identify edge points
    # Edge points are those where the distance to the nearest neighbor is small 
    # AND the dot product of normals is below a threshold (indicating a boundary/edge).
    # This step is complex and depends on specific implementation details of "edge points".
    # For simplicity, we assume edge points are identified by a radius query on the surface.
    # Note: The exact algorithm for edge point identification is not fully specified in [S1, unofficial] 
    # beyond using query_radius and dot product threshold. 
    # We will skip detailed edge point identification logic as it requires specific 
    # surface representation (e.g., mesh edges) which is not provided in the point cloud input.
    # [G] In a full implementation, one would identify edge points from the mesh topology 
    # or by detecting points with high curvature or specific normal variations.
    
    # 7. Compute ECD and EF1 on edge points
    # Since edge points are not explicitly identified in this point-cloud-only version,
    # we return NaN or 0 for these metrics.
    ecd = np.nan
    ef1 = np.nan
    
    # 8. Calculate aspect ratio and radius ratio
    # These require mesh topology (triangles), not just points.
    # [G] If inputs are meshes, these would be computed using pyvista or similar.
    aspect_ratio = np.nan
    radius_ratio = np.nan
    
    # 9. Compute angle regularity
    # Requires mesh topology.
    angle_regularity = np.nan
    
    # 10. Compute self-intersection percentage
    # Requires mesh topology.
    self_intersection_percentage = np.nan
    
    return {
        'cd1': cd1,
        'cd2': cd2,
        'nc': nc,
        'f1': f1,
        'ecd': ecd,
        'ef1': ef1,
        'aspect_ratio': aspect_ratio,
        'radius_ratio': radius_ratio,
        'angle_regularity': angle_regularity,
        'self_intersection_percentage': self_intersection_percentage
    }
```

**Edge Cases and Pitfalls:**
- **Precision:** Floating point precision issues in dot products and distance calculations [S1, unofficial].
- **Boundaries:** Edge metrics (ECD, EF1) require clear definition of "edge points," which is ambiguous for point clouds without topology [S1, unofficial].
- **Degenerate Input:** Empty point clouds or meshes with no triangles will cause division by zero or invalid metrics [S1, unofficial][S2].
- **Visibility:** Occluded or unseen geometry must be explicitly culled; otherwise, error metrics will be inflated by regions where the network output is unpredictable or where ground truth contains hidden surfaces [S3].

## Subtopics

### Mesh Quality Evaluation
A metric to evaluate triangle shape quality, preferring well-shaped triangles over long and skinny ones [S2]. The aspect ratio of triangle $t=abc$ is $\rho(t) = \lambda_2 / \lambda_1$, where $\lambda_1 \ge \lambda_2$ are the greatest eigenvalues of the inertia matrix of $t$ [S2]. $\rho(t) = 1$ if the triangle is equilateral ($\lambda_1 = \lambda_2$) and $\rho(t) = 0$ if it is degenerate ($\lambda_2 = 0$) [S2].

### Evaluation Pipeline
A pipeline to compute Chamfer distance (C-ℓ1), intersection-over-union (IoU), normal consistency (NC), and F-score on reconstructed surfaces visible in input color/depth streams [S3]. It involves subdividing all output meshes to ensure maximum edge length < 1.5 cm, detecting visible vertices using ground truth camera trajectory, culling triangles with zero visible vertices, sampling culled geometry at a density of 1 point per cm², and voxelize reconstruction using 5 cm edge-length voxels for IoU and F-score computation [S3].

## Operating characteristics
- **Time Complexity:** Dominated by KDTree construction and nearest neighbor queries, which are $O(N \log N)$ for construction and $O(\log N)$ per query [S1, unofficial].
- **Memory Complexity:** $O(N)$ for storing points and KDTree [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of points for queries, but KDTree construction is super-linear [S1, unofficial].
- **Strengths:** Comprehensive set of metrics including edge-specific and geometric quality measures [S1, unofficial]. Visibility-aware evaluation prevents inflation of errors in unseen regions [S3].
- **Limitations:** Edge metrics (ECD, EF1) and geometric quality metrics (aspect ratio, etc.) require mesh topology, not just point clouds [S1, unofficial]. Computation can be expensive for large point clouds [S1, unofficial].
- **When to use:** When evaluating the quality of reconstructed meshes or point clouds, especially when edge fidelity and geometric shape are important [S1, unofficial][S3].
- **When not to use:** When only point cloud density or occupancy is of interest, or when mesh topology is not available and edge metrics are required [S1, unofficial].

## Background
Originated in the context of evaluating 3D reconstruction and mesh quality, with specific implementations in projects like TetWeave [S1, unofficial] and Neural RGB-D Surface Reconstruction [S3]. The aspect ratio metric has roots in computational geometry and mesh generation literature [S2].

## Sources
[S1] AlexandreBinninger/TetWeave, https://github.com/AlexandreBinninger/TetWeave, fetched 2026-10-27 (unofficial)
[S2] Isosurface extraction and interpretation on very large datasets in geophysics, https://inria.hal.science/inria-00277070v1, fetched 2026-10-26
[S3] Neural RGB-D Surface Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00619, fetched 2026-10-26
