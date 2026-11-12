# Mesh/Point Cloud Evaluation Metrics

Also called: F-score, F1 score, Mesh Quality Evaluation Metrics

## Overview
Mesh/Point Cloud Evaluation Metrics provide a comprehensive quantitative assessment of the geometric fidelity and topological quality of reconstructed 3D surfaces against ground truth data [S1, S2, unofficial]. While standard metrics like Chamfer Distance measure point-wise proximity, this method extends evaluation to include Normal Consistency, Edge-specific Chamfer distances, and rigorous mesh element quality indicators (aspect ratios, self-intersections) [S2, unofficial]. It differs from its siblings by offering a holistic suite of scalar metrics that cover both geometric accuracy (distance-based) and mesh validity (shape-based), whereas [Chamfer Distance](2446-chamfer-distance.md) focuses solely on point proximity and [Neighbor Counting](2905-neighbor-counting.md) focuses on density estimation.

## How it works
The method evaluates a predicted mesh (or point cloud) against a ground truth mesh by computing distances, normal alignments, and geometric element properties.

**Inputs and Outputs:**
*   **Input:** Predicted mesh/point cloud ($P_{pred}$), Ground truth mesh/point cloud ($P_{GT}$).
*   **Output:** A dictionary of metrics including Chamfer Distance (CD), Normal Consistency (NC), F1 Score, Edge Chamfer Distance (ECD), Edge F1 (EF1), Aspect Ratios, and Self-Intersection counts.

**Data Structures:**
*   **KDTree:** Used for efficient nearest-neighbor queries. Two trees are built: one for $P_{pred}$ and one for $P_{GT}$.
*   **Point Sets:** $P_{pred}$ and $P_{GT}$ are typically sampled to a fixed number of points (default 100,000) to ensure consistent evaluation [S2, unofficial].
*   **Normals:** Unit normal vectors associated with each point or mesh vertex.

**Parameters:**
*   `f1_threshold`: Distance threshold for F1 precision/recall calculation. Default: 0.001 [S2, unofficial].
*   `ef1_threshold`: Distance threshold for Edge F1 calculation. Default: 0.005 [S2, unofficial].
*   `ef1_radius`: Radius for detecting edge points. Default: 0.004 [S2, unofficial].
*   `ef1_dotproduct_threshold`: Dot product threshold for edge classification. Default: 0.2 [S2, unofficial].
*   `num_points`: Number of points to sample for evaluation. Default: 100,000 [S2, unofficial].

**Steps:**

1.  **Sampling:** If inputs are meshes, sample $N$ points uniformly from the surface to create point sets $P_{pred}$ and $P_{GT}$. If inputs are point clouds, ensure they are upsampled or downsampled to match `num_points` [S2, unofficial].
2.  **KDTree Construction:** Build a KDTree for $P_{pred}$ and a KDTree for $P_{GT}$ [S2, unofficial].
3.  **Chamfer Distance (CD) Calculation:**
    *   For each point $p \in P_{pred}$, find the nearest neighbor $q \in P_{GT}$ using the $P_{GT}$ KDTree. Compute distance $d_{pred \to gt} = \|p - q\|$.
    *   For each point $q \in P_{GT}$, find the nearest neighbor $p \in P_{pred}$ using the $P_{pred}$ KDTree. Compute distance $d_{gt \to pred} = \|q - p\|$.
    *   Compute $CD_{L1} = \frac{1}{|P_{pred}|} \sum d_{pred \to gt} + \frac{1}{|P_{GT}|} \sum d_{gt \to pred}$ [S1, unofficial].
    *   Compute $CD_{L2}$ similarly using squared distances or L2 norms [S2, unofficial].
4.  **Normal Consistency (NC) Calculation:**
    *   For each point $p \in P_{pred}$, retrieve its normal $n_p$ and the normal $n_q$ of its nearest neighbor $q \in P_{GT}$.
    *   Compute the dot product: $dot = \frac{n_p \cdot n_q}{\|n_p\| \|n_q\|}$ [S2, unofficial].
    *   Average the dot products over all points.
5.  **F-Score Calculation:**
    *   Compute Precision: The fraction of points in $P_{pred}$ whose nearest neighbor in $P_{GT}$ is within `f1_threshold`.
        $$Precision = \frac{\sum_{p \in P_{pred}} \mathbb{I}(\min_{q \in P_{GT}} \|p - q\| < \text{f1\_threshold})}{|P_{pred}|}$$
    *   Compute Recall: The fraction of points in $P_{GT}$ whose nearest neighbor in $P_{pred}$ is within `f1_threshold`.
        $$Recall = \frac{\sum_{q \in P_{GT}} \mathbb{I}(\min_{p \in P_{pred}} \|q - p\| < \text{f1\_threshold})}{|P_{GT}|}$$
    *   Compute F-Score:
        $$F\text{-score} = \frac{2 \cdot Precision \cdot Recall}{Precision + Recall}$$
        [S1, S2, unofficial].
6.  **Edge Point Identification:**
    *   Identify "edge points" in both $P_{pred}$ and $P_{GT}$. A point is considered an edge point if it has fewer neighbors within `ef1_radius` than a surface point, or by classifying based on local geometry. Specifically, use `query_radius` with `ef1_radius` and check the dot product of normals between the point and its neighbors. If the dot product is below `ef1_dotproduct_threshold`, it indicates a sharp feature/edge [S2, unofficial].
7.  **Edge Chamfer Distance (ECD) and Edge F1 (EF1):**
    *   Compute Chamfer Distance and F1 Score (using `ef1_threshold`) only on the subset of points identified as edge points in Step 6 [S2, unofficial].
8.  **Mesh Quality Metrics (for Mesh Inputs):**
    *   **Aspect Ratio:** For each triangle $t$ with vertices $a, b, c$, compute the centroid $\hat{t}$. Compute the inertia matrix $M_t = \frac{1}{3} [(a - \hat{t})(a - \hat{t})^T + (b - \hat{t})(b - \hat{t})^T + (c - \hat{t})(c - \hat{t})^T]$ [S4]. Find the greatest eigenvalues $\lambda_1 \ge \lambda_2$. Aspect ratio $\rho(t) = \lambda_2 / \lambda_1$ [S4].
    *   **Radius Ratio:** Computed via libraries like `pyvista` [S2, unofficial].
    *   **Angle Regularity:** Computed via libraries like `trimesh` [S2, unofficial].
    *   **Self-Intersection:** Count the number of self-intersecting triangles [S3, unofficial].
    *   **Flipped Elements:** Count flipped triangles (`triFlip`) and flipped tetrahedra (`tetFlip`) if volumetric data is available [S3, unofficial].
9.  **Reporting Aggregation:**
    *   For a dataset of meshes, report the **average** of the mean aspect ratios (triAR and tetAR) across all meshes.
    *   Report the **median** of the minimum and maximum aspect ratios across all meshes to remove extreme outliers [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.spatial import KDTree

def compute_mesh_metrics(pred_points, pred_normals, gt_points, gt_normals, 
                         f1_threshold=0.001, ef1_threshold=0.005, 
                         ef1_radius=0.004, ef1_dot_threshold=0.2):
    """
    Computes Chamfer Distance, Normal Consistency, F1 Score, and Edge Metrics.
    Assumes points are already sampled to equal length.
    """
    # 1. Build KDTree
    tree_pred = KDTree(pred_points)
    tree_gt = KDTree(gt_points)
    
    # 2. Nearest Neighbors
    dist_pred_to_gt, idx_pred_to_gt = tree_pred.query(gt_points, k=1)
    dist_gt_to_pred, idx_gt_to_pred = tree_gt.query(pred_points, k=1)
    
    # 3. Chamfer Distance (L1)
    cd_l1 = np.mean(dist_pred_to_gt) + np.mean(dist_gt_to_pred)
    
    # 4. Normal Consistency
    # Normalize normals just in case
    pred_n = pred_normals / (np.linalg.norm(pred_normals, axis=1, keepdims=True) + 1e-8)
    gt_n = gt_normals / (np.linalg.norm(gt_normals, axis=1, keepdims=True) + 1e-8)
    
    # Get normals of nearest neighbors
    neighbor_normals_gt = pred_n[idx_gt_to_pred]
    neighbor_normals_pred = gt_n[idx_pred_to_pred]
    
    # Dot product
    dot_product = np.sum(pred_n * neighbor_normals_gt, axis=1)
    nc = np.mean(dot_product)
    
    # 5. F-Score
    # Precision: fraction of pred points close to GT
    precision = np.mean(dist_gt_to_pred < f1_threshold)
    # Recall: fraction of GT points close to Pred
    recall = np.mean(dist_pred_to_gt < f1_threshold)
    
    if precision + recall == 0:
        f1_score = 0.0
    else:
        f1_score = 2 * precision * recall / (precision + recall)
        
    # 6. Edge Detection (Simplified heuristic based on source [S2, unofficial])
    # In practice, this requires identifying boundary/feature points.
    # Here we assume edge masks are provided or computed via local geometry.
    # For the purpose of this implementation, we assume binary masks for edges.
    # If not provided, this step is marked as requiring external geometry processing.
    # [G] Edge detection logic is complex; standard implementation uses 
    # query_radius to find neighbors and checks normal variance or count.
    
    # Placeholder for edge metrics if edge masks were available:
    # edge_pred_mask, edge_gt_mask = detect_edges(pred_points, pred_normals, ef1_radius, ef1_dot_threshold)
    # edge_pred_pts = pred_points[edge_pred_mask]
    # edge_gt_pts = gt_points[edge_gt_mask]
    # ... compute ECD and EF1 on these subsets ...
    
    return {
        "cd_l1": cd_l1,
        "nc": nc,
        "f1_score": f1_score,
        "precision": precision,
        "recall": recall
    }

def compute_aspect_ratio(vertices, faces):
    """
    Computes mean aspect ratio for a mesh.
    """
    # Get vertices for each face
    v0 = vertices[faces[:, 0]]
    v1 = vertices[faces[:, 1]]
    v2 = vertices[faces[:, 2]]
    
    # Centroids
    centroids = (v0 + v1 + v2) / 3.0
    
    # Vectors from centroid
    d0 = v0 - centroids
    d1 = v1 - centroids
    d2 = v2 - centroids
    
    # Inertia matrix per triangle (3x3)
    # M = 1/3 * (d0*d0^T + d1*d1^T + d2*d2^T)
    # We compute eigenvalues for each triangle
    
    aspect_ratios = []
    for i in range(len(faces)):
        M = (np.outer(d0[i], d0[i]) + np.outer(d1[i], d1[i]) + np.outer(d2[i], d2[i])) / 3.0
        eigenvalues = np.linalg.eigvalsh(M)
        # eigenvalues are sorted ascending
        lambda2 = eigenvalues[-1]
        lambda1 = eigenvalues[-2]
        
        if lambda1 == 0:
            ar = 0.0
        else:
            ar = lambda2 / lambda1
        aspect_ratios.append(ar)
        
    return np.mean(aspect_ratios)
```

## Subtopics

### [Evaluation Pipeline](9358-evaluation-pipeline.md)
This pipeline computes Chamfer distance ($C_{\ell1}$), intersection-over-union (IoU), normal consistency (NC), and F-score on reconstructed surfaces that are visible in the input color/depth streams. It differs from standard mesh evaluation by explicitly culling occluded or unseen geometry to prevent error inflation from regions where the network output is unpredictable or where ground truth contains hidden surfaces. It builds on [Mesh/Point Cloud Evaluation Metrics](4797-mesh-point-cloud-evaluation-metrics.md) but restricts the comparison domain to the viewing frusta of the input cameras.

### [Evaluation Metrics](9899-evaluation-metrics.md)
Computes quantitative assessment of rendering quality against ground truth using Peak Signal-to-Noise Ratio (PSNR) and Structural Similarity Index Measure (SSIM). Additionally, it assesses mesh quality using Chamfer Distance (CD), F1 score, and Normal Consistency (NC) to evaluate geometric fidelity. It differs from the [Evaluation Pipeline](9358-evaluation-pipeline.md) by focusing specifically on the mathematical computation of these scalar metrics rather than the orchestration of data loading and training loops.

### [Cloud-to-Mesh Accuracy Metrics](10560-cloud-to-mesh-accuracy-metrics.md)
Cloud-to-mesh accuracy metrics quantify how closely a reconstructed point cloud reflects ground truth locations by computing the displacements between each point in the compared point cloud and the nearest facet in the reference mesh. It serves as a precision measure, distinguishing itself from completeness (recall) metrics by focusing on the correctness of the reconstructed geometry rather than its coverage. It is a component of the broader [Mesh/Point Cloud Evaluation Metrics](4797-mesh-point-cloud-evaluation-metrics.md) family.

### [SDF Consistency Metrics](13280-sdf-consistency-metrics.md)
SDF Consistency Metrics evaluate how closely a reconstructed mesh matches the trained Signed Distance Function (SDF) field by measuring the network's SDF output at points on the mesh surface and at its vertices. Unlike geometric metrics that compare two meshes, this method assesses the internal consistency of the implicit representation, ensuring the extracted geometry aligns with the learned field. It is distinct from [Cloud-to-Mesh Accuracy Metrics](10560-cloud-to-mesh-accuracy-metrics.md) which measure distance to ground truth, and [Evaluation Metrics](9899-evaluation-metrics.md) which provide a broader suite of quality indicators.

### Mesh Quality Evaluation
A metric to evaluate triangle shape quality, preferring well-shaped triangles over long and skinny ones [S4]. The aspect ratio of triangle $t=abc$ is $\rho(t) = \lambda_2 / \lambda_1$, where $\lambda_1 \ge \lambda_2$ are the greatest eigenvalues of the inertia matrix of $t$ [S4]. $\rho(t) = 1$ if the triangle is equilateral ($\lambda_1 = \lambda_2$) and $\rho(t) = 0$ if it is degenerate ($\lambda_2 = 0$) [S4].

### Shape Novelty Verification
To verify generated models are new shapes and not memorized training data: Given a generated mesh, retrieve the most similar mesh in the training set by computing the Chamfer distance of point cloud samples from the generated mesh against all meshes in the training set [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by KDTree construction ($O(N \log N)$) and nearest-neighbor queries ($O(N \log N)$) for Chamfer Distance and F1 Score [S2, unofficial]. Edge detection and mesh quality metrics (eigenvalue decomposition) add overhead proportional to the number of faces/points.
*   **Memory Complexity:** $O(N)$ for storing point sets and KDTree structures [S2, unofficial].
*   **Performance:** The method is computationally efficient for point cloud comparisons. Mesh quality metrics (aspect ratio, self-intersection) require more intensive geometric processing [S3, unofficial].
*   **Strengths:** Provides a multi-faceted view of quality (geometric + topological). The F1 score is robust to noise compared to raw Chamfer Distance. Edge metrics focus on critical features.
*   **Limitations:** Sensitive to sampling density; uneven sampling can skew Chamfer Distance. F1 score depends heavily on the choice of `f1_threshold`. Self-intersection detection can be expensive for high-resolution meshes [S3, unofficial].
*   **When to use:** When evaluating 3D reconstruction algorithms (NeRF, SDF, Mesh generation) where both geometric accuracy and mesh validity are important.
*   **When not to use:** When only view-synthesis quality (PSNR/SSIM) is of interest, or when computational resources are extremely limited and only a single distance metric is required.

## Background
The F-score and Chamfer Distance are standard metrics in 3D computer vision and graphics for evaluating point cloud and mesh reconstruction accuracy [S1, unofficial]. The comprehensive suite including Edge Chamfer Distance and Normal Consistency was popularized in recent neural rendering and implicit surface reconstruction papers [S2, unofficial]. Mesh quality metrics like aspect ratio and tetrahedron validity are rooted in finite element analysis and computational geometry [S3, unofficial][S4]. The practice of using median min/max aspect ratios to report mesh quality while averaging mean aspect ratios was introduced to mitigate the impact of extreme outliers in large-scale evaluations [S3, unofficial].

## Sources
[S1] From reconstruction to reasoning : neural radiance fields for comprehensive 3D scene understanding, https://theses.hal.science/tel-05693528v1, fetched 2026-10-26 (unofficial)
[S2] AlexandreBinninger/TetWeave, https://github.com/AlexandreBinninger/TetWeave, fetched 2026-10-27 (unofficial)
[S3] Neural Volumetric Mesh Generator, https://arxiv.org/abs/2210.03158v1, fetched 2026-10-27 (unofficial)
[S4] Isosurface extraction and interpretation on very large datasets in geophysics, https://inria.hal.science/inria-00277070v1, fetched 2026-10-26
