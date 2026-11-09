# Evaluation Metrics

Also called: PSNR and SSIM Calculation, Rendering Quality Assessment

## Overview
Computes quantitative assessment of rendering quality against ground truth using Peak Signal-to-Noise Ratio (PSNR) and Structural Similarity Index Measure (SSIM) [S2, unofficial]. Additionally, it assesses mesh quality using Chamfer Distance (CD), F1 score, and Normal Consistency (NC) to evaluate geometric fidelity [S1]. It differs from the [Evaluation Pipeline](9358-evaluation-pipeline.md) by focusing specifically on the mathematical computation of these scalar metrics rather than the orchestration of data loading and training loops.

## How it works
**Inputs:**
1.  Rendered image tensor $I_{render}$ and Ground Truth image tensor $I_{gt}$ [S2, unofficial].
2.  Reconstructed mesh $M$ and input point cloud $P$ for mesh evaluation [S1].

**Outputs:**
1.  Scalar values for PSNR, SSIM, CD, F1, and NC [S1][S2, unofficial].

**Data Structures:**
*   Image tensors: Shape $(H, W, C)$ or $(B, H, W, C)$, values normalized to $[0, 1]$ [S2, unofficial].
*   Point cloud: Set of 3D coordinates $\{p_i\}$ [S1].
*   Mesh: Set of faces $F$ and vertices $V$ [S1].

**Parameters:**
*   `channel_axis`: Set to $-1$ for RGB channels in the last dimension [S2, unofficial].
*   `data_range`: Set to $1.0$ for normalized $[0,1]$ image data [S2, unofficial].
*   F1 threshold $\tau$: Set to $0.003$ to ensure sensitivity to fine details [S1].
*   CD scaling factor: All reported Chamfer distances are scaled by $10^5$ [S1].
*   Sampling count: $1,000,000$ points sampled for mesh evaluation [S1].

**Steps:**

1.  **Image Quality Assessment (PSNR/SSIM):**
    *   Ensure $I_{render}$ and $I_{gt}$ are aligned in shape and data type [S2, unofficial].
    *   Compute Mean Squared Error (MSE) between $I_{render}$ and $I_{gt}$:
        $$ \text{MSE} = \frac{1}{N} \sum_{i=1}^{N} (I_{render, i} - I_{gt, i})^2 $$
        where $N$ is the total number of pixels.
    *   Compute PSNR:
        $$ \text{PSNR} = 10 \cdot \log_{10} \left( \frac{\text{MAX}_I^2}{\text{MSE}} \right) $$
        where $\text{MAX}_I = 1.0$ given the normalized range [S2, unofficial].
    *   Compute SSIM using a sliding window approach (typically $11 \times 11$) comparing luminance, contrast, and structure between local windows of $I_{render}$ and $I_{gt}$ [S2, unofficial].

2.  **Mesh Quality Assessment (CD, F1, NC):**
    *   Sample $N_{sample} = 1,000,000$ points $P_{sample}$ uniformly from the surface of the reconstructed mesh $M$ [S1].
    *   Perform face-to-point projection using the Fast Closest Point with Weights (FCPW) method [S1, S30]. For each point $p \in P_{sample}$, find the closest point $q$ on the input point cloud $P$ and the closest face on $M$ [S1].
    *   Compute Chamfer Distance (CD):
        $$ \text{CD} = \frac{1}{N_{sample}} \sum_{p \in P_{sample}} \min_{q \in P} \|p - q\|^2 + \frac{1}{|P|} \sum_{q \in P} \min_{p \in P_{sample}} \|q - p\|^2 $$
        Scale the final CD value by $10^5$ [S1].
    *   Compute F1 Score:
        *   Calculate the distance from each point in $P_{sample}$ to $P$.
        *   Count points where distance $< \tau$ ($0.003$) as True Positives (TP).
        *   Count points in $P$ where distance to $P_{sample} < \tau$ as True Positives for the reverse direction.
        *   Compute Precision and Recall based on these thresholds.
        *   $F1 = 2 \cdot \frac{\text{Precision} \cdot \text{Recall}}{\text{Precision} + \text{Recall}}$ [S1].
    *   Compute Normal Consistency (NC):
        *   Compare the normal vector of the mesh face at the projection point with the normal vector of the input point cloud (if available) or the estimated normal from the local neighborhood [S1].
        *   Average the dot product of corresponding normals.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch

def compute_psnr_ssim(rendered: torch.Tensor, ground_truth: torch.Tensor) -> tuple:
    """
    Computes PSNR and SSIM for rendered vs ground truth images.
    Assumes inputs are in [0, 1] range with shape (H, W, C) or (B, H, W, C).
    """
    # MSE Calculation
    mse = torch.mean((rendered - ground_truth) ** 2)
    
    # PSNR Calculation
    # MAX_I is 1.0 for normalized data
    psnr = 10 * torch.log10(1.0 / mse)
    
    # SSIM Calculation
    # Using a standard implementation for SSIM with window size 11
    # This requires a helper function for SSIM which is standard in libraries like skimage
    # Here we assume a generic ssim function exists or is implemented
    ssim_val = compute_ssim(rendered, ground_truth, channel_axis=-1, data_range=1.0)
    
    return psnr, ssim_val

def compute_mesh_metrics(mesh_points: torch.Tensor, input_points: torch.Tensor) -> dict:
    """
    Computes CD, F1, and NC for mesh quality.
    mesh_points: Sampled points from mesh surface (1M points).
    input_points: Ground truth point cloud.
    """
    # Constants
    threshold = 0.003
    scale_factor = 1e5
    
    # 1. Chamfer Distance
    # Compute pairwise distances or use efficient nearest neighbor search
    # dist_mesh_to_gt: min distance from each mesh point to input_points
    # dist_gt_to_mesh: min distance from each input point to mesh_points
    dist_mesh_to_gt = compute_min_distance(mesh_points, input_points)
    dist_gt_to_mesh = compute_min_distance(input_points, mesh_points)
    
    cd = torch.mean(dist_mesh_to_gt**2) + torch.mean(dist_gt_to_mesh**2)
    cd_scaled = cd * scale_factor
    
    # 2. F1 Score
    # Precision: fraction of mesh points close to input points
    precision_mask = dist_mesh_to_gt < threshold
    precision = torch.sum(precision_mask) / len(precision_mask)
    
    # Recall: fraction of input points close to mesh points
    recall_mask = dist_gt_to_mesh < threshold
    recall = torch.sum(recall_mask) / len(recall_mask)
    
    if precision + recall == 0:
        f1 = 0.0
    else:
        f1 = 2 * (precision * recall) / (precision + recall)
        
    # 3. Normal Consistency
    # Assuming normals are available or estimated
    # This step requires access to mesh normals and point cloud normals
    # Placeholder for specific NC implementation details not fully provided in source
    # Typically: mean(dot_product(mesh_normals, gt_normals))
    nc = compute_normal_consistency(mesh_points, input_points) 
    
    return {
        "cd_scaled": cd_scaled.item(),
        "f1": f1.item(),
        "nc": nc.item()
    }

def compute_min_distance(points_a: torch.Tensor, points_b: torch.Tensor) -> torch.Tensor:
    """
    Computes minimum distance from each point in points_a to any point in points_b.
    Uses efficient nearest neighbor search (e.g., KDTree or CUDA-based).
    """
    # Implementation detail: Use a library like scipy.spatial.cKDTree or torch_cluster
    # For pure Python/NumPy without external libs, this would be O(N*M) which is slow.
    # Given the context of "capable programmer", we assume access to efficient NN search.
    # Here we describe the logic:
    # 1. Build spatial index for points_b.
    # 2. Query nearest neighbor for each point in points_a.
    # 3. Return distances.
    pass # Implementation depends on specific efficient NN library used in the pipeline
```

## Subtopics

### Real-world Dataset Evaluation
Evaluates the method on real-world scenes from the BlendedMVS and Bag of Chips datasets, which provide images and depth maps reconstructed by MVS methods or RGBD cameras [S3]. For BlendedMVS, the last ten images are used as the test set; for Bag of Chips, the last 1/5 of the images are used [S3]. Performance degrades on this data due to varying exposures, rough geometry, and insufficient background in original images for ambient illumination reconstruction [S3].

## Operating characteristics
*   **Time Complexity:** Image metrics (PSNR/SSIM) are $O(N)$ where $N$ is the number of pixels. Mesh metrics (CD/F1/NC) depend on the efficiency of the nearest neighbor search between the sampled points and the input cloud, typically $O(N \log N)$ with spatial indexing [S1].
*   **Memory Complexity:** Requires storing the sampled points ($10^6$ points) and the input point cloud [S1].
*   **Strengths:** Provides standardized, quantitative comparison for both rendering quality (PSNR/SSIM) and geometric accuracy (CD/F1/NC) [S1][S2, unofficial].
*   **Limitations:** Sensitive to fine details via the F1 threshold ($0.003$) [S1]. Real-world evaluation is limited by dataset artifacts like exposure variation and lack of background information [S3].

## Background
The mesh evaluation metrics (CD, F1, NC) are described in the context of DCCVT (Differentiable Clipped Centroidal Voronoi Tessellation) [S1]. The image quality metrics (PSNR, SSIM) are standard in 3D Gaussian Splatting evaluation pipelines [S2, unofficial]. Real-world dataset evaluation is discussed in the context of Neural Illumination Fields (NeIF) [S3].

## Sources
[S1] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
[S2] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S3] A Pre-convolved Representation for Plug-and-Play Neural Illumination Fields, https://doi.org/10.1609/aaai.v38i7.28618, fetched 2026-10-29
