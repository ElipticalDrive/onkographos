# Evaluation Metrics for 3D Reconstruction

Also called: 3D Reconstruction Evaluation Metrics, F-Score, INR-SLAM Metrics, Point Cloud Quality Metrics, Reconstruction and Pose Metrics

## Overview
This method defines a comprehensive protocol for evaluating the geometric, photometric, and procedural quality of 3D reconstruction systems, particularly those involving Implicit Neural Representations (INRs) and SLAM [S2]. It addresses the need to quantify both static reconstruction fidelity and dynamic tracking performance [S2]. It distinguishes itself from [Mesh Quality Evaluation](8534-mesh-quality-evaluation.md) by focusing on point-cloud-based geometric distances and SLAM-specific trajectory errors, and from [Evaluation Pipeline](9358-evaluation-pipeline.md) by providing the specific mathematical definitions for accuracy, completeness, and photometric metrics [S2, S3].

## How it works
**Inputs and Outputs**
- **Inputs**: Predicted mesh (or implicit field), ground truth mesh, predicted depth maps, ground truth depth maps, camera poses, and ground truth camera poses [S2, S3].
- **Outputs**: Scalar metrics for Accuracy, Completion, Completion Ratio, ATE RMSE, PSNR, Depth L1, Normal Consistency, Precision, Recall, F-score, and Chamfer-L1 Distance [S2, S3].

**Data Structures**
- $P$: Sampled point cloud from the predicted mesh [S3].
- $\bar{P}$: Sampled point cloud from the ground truth mesh [S3].
- $n_p, n_{\bar{p}}$: Normal vectors computed from $P$ and $\bar{P}$ respectively [S3].
- $C^n_{pred}, C^n_{gt}$: The $n$-th predicted and ground truth image/depth map [S3].
- $N_I$: Number of images used for evaluation [S3].

**Preprocessing**
Prior to metric calculation, mesh culling is conducted to remove invalid geometry [S2].

**Metrics Calculation**
Lower values denote superior performance for accuracy, completeness, and chamfer L1 distance, whereas higher values are preferable for precision, recall, F-score, normal consistency, and PSNR [S3].

1.  **Geometric Accuracy (Acc)**: The mean distance from predicted points to the ground truth surface.
    $$ \text{Acc} = \frac{1}{|P|} \sum_{p \in P} \min_{\bar{p} \in \bar{P}} \| p - \bar{p} \|_1 $$
    Lower values indicate better performance [S3].

2.  **Geometric Completion (Comp)**: The mean distance from ground truth points to the predicted surface.
    $$ \text{Comp} = \frac{1}{|\bar{P}|} \sum_{\bar{p} \in \bar{P}} \min_{p \in P} \| \bar{p} - p \|_1 $$
    Lower values indicate better performance [S3].

3.  **Completion Ratio (Comp. [%])**: The proportion of extracted meshes where the Completion value is smaller than 5 cm.
    $$ \text{Comp. [\%]} = \frac{1}{N_{meshes}} \sum_{i=1}^{N_{meshes}} \mathbb{I}(\text{Comp}_i < 5 \text{ cm}) \times 100 $$
    Higher values indicate better performance [S2].

4.  **Precision (Prec)**: The proportion of predicted points within a threshold (0.05) of the ground truth.
    $$ \text{Prec} = \frac{1}{|P|} \sum_{p \in P} \mathbb{I}\left( \min_{\bar{p} \in \bar{P}} \| p - \bar{p} \|_1 < 0.05 \right) $$
    Higher values indicate better performance [S3].

5.  **Recall (Recall)**: The proportion of ground truth points within a threshold (0.05) of the predicted surface.
    $$ \text{Recall} = \frac{1}{|\bar{P}|} \sum_{\bar{p} \in \bar{P}} \mathbb{I}\left( \min_{p \in P} \| \bar{p} - p \|_1 < 0.05 \right) $$
    Higher values indicate better performance [S3].

6.  **F-score**: The harmonic mean of Precision and Recall.
    $$ \text{F-score} = \frac{2 \times \text{Prec} \times \text{Recall}}{\text{Prec} + \text{Recall}} $$
    Higher values indicate better performance [S1, S3].

7.  **Normal Consistency**: The mean dot product of predicted and ground truth normals at corresponding points.
    $$ \text{Normal Consistency} = \frac{1}{|P|} \sum_{p \in P} n_p^T n_{\bar{p}} $$
    where $\bar{p} = \arg\min_{\bar{p} \in \bar{P}} \| p - \bar{p} \|_1$ [S3]. Higher values indicate better performance [S3].

8.  **Chamfer-L1 Distance**: The sum of Accuracy and Completion.
    $$ \text{Chamfer-L1} = \text{Acc} + \text{Comp} $$
    Lower values indicate better performance [S3].

9.  **Camera Trajectory Accuracy (ATE RMSE)**: The Root Mean Square Error of the camera trajectory.
    $$ \text{ATE [cm] RMSE} = \sqrt{\frac{1}{N} \sum_{i=1}^{N} \| T_{pred, i} - T_{gt, i} \|^2} $$
    Lower values indicate better performance [S2].

10. **Photometric Precision (PSNR)**: Peak Signal-to-Noise Ratio of the estimated depth.
    $$ \text{PSNR [dB]} = \frac{1}{N_I} \sum_{n=1}^{N_I} 20 \cdot \log_{10} \left( \frac{1}{\sqrt{\text{MSE}(C^n_{pred}, C^n_{gt})}} \right) $$
    Higher values indicate better performance [S3].

11. **Depth L1**: The L1 term of the estimated depth compared to ground truth.
    $$ \text{Depth L1 [cm]} = \frac{1}{N_I} \sum_{n=1}^{N_I} \| C^n_{pred} - C^n_{gt} \|_1 $$
    Lower values indicate better performance [S2].

12. **Efficiency Metrics**: Average update time per input frame for mapping and tracking over the whole sequence [S2].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def evaluate_reconstruction(P, P_bar, normals_p, normals_p_bar, 
                            C_pred, C_gt, poses_pred, poses_gt, 
                            N_I, threshold=0.05, comp_threshold_cm=5.0):
    """
    P: list of (x,y,z) predicted points
    P_bar: list of (x,y,z) ground truth points
    normals_p: list of (nx,ny,nz) predicted normals
    normals_p_bar: list of (nx,ny,nz) ground truth normals
    C_pred: list of 2D arrays (depth maps)
    C_gt: list of 2D arrays (depth maps)
    poses_pred: list of 4x4 matrices (predicted poses)
    poses_gt: list of 4x4 matrices (ground truth poses)
    """
    
    # 1. Geometric Accuracy (Acc)
    acc = sum(min(sum(abs(p[i] - pb[i]) for i in range(3)) for pb in P_bar) for p in P) / len(P)
    
    # 2. Geometric Completion (Comp)
    comp = sum(min(sum(abs(pb[i] - p[i]) for i in range(3)) for p in P) for pb in P_bar) / len(P_bar)
    
    # 3. Completion Ratio
    comp_ratio = 100.0 if comp < comp_threshold_cm else 0.0
    
    # 4. Precision
    prec = sum(1 for p in P if min(sum(abs(p[i] - pb[i]) for i in range(3)) for pb in P_bar) < threshold) / len(P)
    
    # 5. Recall
    recall = sum(1 for pb in P_bar if min(sum(abs(pb[i] - p[i]) for i in range(3)) for p in P) < threshold) / len(P_bar)
    
    # 6. F-score
    f_score = 2 * (prec * recall) / (prec + recall) if (prec + recall) > 0 else 0.0
        
    # 7. Normal Consistency
    closest_normals_p_bar = []
    for p in P:
        min_dist = float('inf')
        closest_nb = None
        for i, pb in enumerate(P_bar):
            dist = sum(abs(p[i] - pb[i]) for i in range(3))
            if dist < min_dist:
                min_dist = dist
                closest_nb = normals_p_bar[i]
        closest_normals_p_bar.append(closest_nb)
    normal_consistency = sum(sum(np[i] * cnb[i] for i in range(3)) for np, cnb in zip(normals_p, closest_normals_p_bar)) / len(P)
    
    # 8. Chamfer-L1
    chamfer_l1 = acc + comp
    
    # 9. ATE RMSE
    pose_errors_sq = sum(sum((poses_pred[k][r][c] - poses_gt[k][r][c])**2 for c in range(4)) for r in range(4)) / len(poses_pred)
    ate_rmse = math.sqrt(pose_errors_sq)
    
    # 10. PSNR
    mse_sum = sum(sum((C_pred[n][r][c] - C_gt[n][r][c])**2 for c in range(len(C_pred[n][0])) for r in range(len(C_pred[n]))) for n in range(N_I))
    total_pixels = N_I * len(C_pred[0]) * len(C_pred[0][0])
    mse_avg = mse_sum / total_pixels
    psnr = 20 * math.log10(1 / math.sqrt(mse_avg)) if mse_avg > 0 else float('inf')
    
    # 11. Depth L1
    depth_l1 = sum(sum(abs(C_pred[n][r][c] - C_gt[n][r][c]) for c in range(len(C_pred[n][0])) for r in range(len(C_pred[n]))) for n in range(N_I)) / total_pixels
    
    return {
        'Acc': acc, 'Comp': comp, 'Comp_Ratio': comp_ratio,
        'Prec': prec, 'Recall': recall, 'F_Score': f_score,
        'Normal_Consistency': normal_consistency, 'Chamfer_L1': chamfer_l1,
        'ATE_RMSE': ate_rmse, 'PSNR': psnr, 'Depth_L1': depth_l1
    }
```

## Subtopics

### [Point Spawning Accuracy Evaluation](19169-point-spawning-accuracy-evaluation.md)
Point Spawning Accuracy Evaluation measures the geometric fidelity of reconstructed point clouds against ground truth data using the F-score metric, specifically within the context of the Tanks&Temples benchmark. It distinguishes itself from photometric metrics by focusing exclusively on spatial alignment, combining precision and recall to penalize both missing geometry and false positives.

### [Cloud-to-Cloud Comparison](22334-cloud-to-cloud-comparison.md)
Cloud-to-Cloud (C2C) comparison is a quantitative evaluation method that assesses the geometric accuracy and alignment of a reconstructed 3D model against a ground truth (GT) reference by computing the relative Euclidean distances between corresponding 3D sample points. It differs from global registration metrics by focusing on the per-point deviation magnitude rather than just the transformation error, and it is distinct from surface-based metrics (like Chamfer distance) by typically operating on discrete point samples rather than continuous meshes.

### Plane Fitting Noise Evaluation
Plane Fitting Noise Evaluation measures reconstruction noise by fitting an ideal mathematical plane to reconstructed surface points on known flat geometries and computing deviation statistics [S4]. Input image counts vary by dataset, with the Stair dataset using identical counts for both methods and the Synthetic dataset using 200 images for Instant-NGP and 24 images for photogrammetry [S4]. On the Stair dataset, both Photogrammetry and NeRF yield an Average RMSE of 0.72 mm, though NeRF shows higher Max/STD on individual planes [S4]. On the Synthetic dataset, Photogrammetry H_plane_01 RMSE is 0.015 mm versus NeRF's 0.070 mm [S4]. Photogrammetry consistently yields lower noise metrics than NeRF, with NeRF RMSE values typically 2–3 times higher than photogrammetric results on flat surfaces [S4].

## Operating characteristics
- **Time Complexity**: Geometric metrics (Acc, Comp, Chamfer) depend on the number of points $|P|$ and $|\bar{P}|$ [G]. Naive pairwise distance computation is $O(|P| \cdot |\bar{P}|)$ [G]. Efficient implementations use spatial indexing (e.g., KD-trees) to reduce this to approximately $O(|P| \log |\bar{P}|)$ [G]. PSNR and Depth L1 are $O(N_I \cdot H \cdot W)$ [G].
- **Memory Complexity**: $O(|P| + |\bar{P}|)$ to store point clouds and normals [G]. $O(N_I \cdot H \cdot W)$ for depth maps [G].
- **Strengths**: Provides a holistic view of reconstruction quality by combining geometric fidelity (Acc/Comp), surface completeness (Recall/Prec), and photometric accuracy (PSNR) [S2, S3]. The inclusion of ATE RMSE allows for direct comparison with SLAM systems [S2].
- **Limitations**: Accuracy and Completion are sensitive to the sampling density of the meshes [G]. The threshold for Completion Ratio (5 cm) is arbitrary and scene-dependent [S2]. Normal consistency assumes aligned normals, which may not hold for all reconstruction methods [G].
- **When to use**: When evaluating neural scene reconstruction, INR-based SLAM, or any method producing a mesh and depth maps [S2, S3].
- **When not to use**: For purely static mesh quality assessment without photometric components (use [Mesh Quality Evaluation](8534-mesh-quality-evaluation.md)) or when only point cloud accuracy is needed without completeness (use [Cloud-to-Mesh Accuracy Metrics](10560-cloud-to-mesh-accuracy-metrics.md)) [G].

## Background
This evaluation protocol was established to benchmark Implicit Neural Representation (INR) based RGB-D SLAM systems [S2]. It formalizes metrics for evaluating neural scene reconstruction methods [S3]. The protocol has been applied to compare NeRF-based reconstructions against traditional photogrammetry, revealing systematic noise differences on flat surfaces [S4].

## Sources
[S1] Multi-View Stereo Network Based on Attention Mechanism and Neural Volume Rendering, https://doi.org/10.3390/electronics12224603, fetched 2026-10-26
[S2] Benchmarking Implicit Neural Representation and Geometric Rendering in Real-Time RGB-D SLAM, https://doi.org/10.1109/cvpr52733.2024.02017, fetched 2026-10-26
[S3] Ray-Distance Volume Rendering for Neural Scene Reconstruction, https://doi.org/10.1007/978-3-031-72630-9_22, fetched 2026-10-26
[S4] A Critical Analysis of NeRF-Based 3D Reconstruction, https://doi.org/10.3390/rs15143585, fetched 2026-10-26
