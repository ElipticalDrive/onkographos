# Normal Consistency

Also called: Mean Cosine Error

## Overview
Normal Consistency (NC) is a metric for 3D surface reconstruction that measures the angular deviation between estimated surface normals and ground truth normals [S1]. It quantifies how well the orientation of the reconstructed surface aligns with the underlying geometry [S1]. Unlike Mesh Reconstruction Evaluation which may focus on vertex/face topology or Standard Rendering Metrics which focus on pixel-wise photometric error, NC specifically isolates geometric orientation accuracy [S1, S4].

## How it works
**Inputs and Outputs:**
- **Inputs:** A set of $N$ points on the surface. For each point $i$, an estimated normal vector $\mathbf{n}_i$ (or $N_i$) and a ground truth normal vector $\mathbf{n}_i^*$ (or $N_i^*$). Both vectors must be normalized (unit length).
- **Output:** A scalar value representing the average angular error, typically expressed as a percentage or in radians/degrees depending on the variant.

**Data Structures:**
- Two arrays of shape $(N, 3)$ containing the estimated and ground truth normals, respectively.
- $N$: The total number of points sampled on the surface.

**Formula:**
The standard Normal Consistency (NC) metric is defined as [S1]:

$$
\text{NC} = \frac{1}{N} \sum_{i=1}^N (1 - \cos(N_i, N_i^*)) \times 100\%
$$

Where:
- $N_i$ is the estimated normal for the $i$-th point.
- $N_i^*$ is the ground truth normal for the $i$-th point.
- $\cos(N_i, N_i^*)$ is the cosine similarity, calculated as the dot product $N_i \cdot N_i^*$ for unit vectors.
- $N$ is the number of points.

An alternative formulation, often called Mean Cosine Error (MCE), computes the mean angle directly [S2]:

$$
\text{MCE} = \frac{1}{N} \sum_{i=1}^N \cos^{-1} (\mathbf{n}_i \cdot \hat{\mathbf{n}}_i)
$$

Where $\mathbf{n}_i$ is the exact normal and $\hat{\mathbf{n}}_i$ is the estimated normal. Note that MCE yields results in radians, while NC yields a percentage where 0% indicates perfect consistency and 100% indicates orthogonal normals [S1, S2].

**Steps:**
1. Sample $N$ points on the surface of the reconstructed mesh.
2. Compute or retrieve the ground truth normal $\mathbf{n}_i^*$ for each point.
3. Compute the estimated normal $\mathbf{n}_i$ for each point. This is often done by computing the gradient of a Signed Distance Function (SDF) at the point and normalizing it, or by averaging face normals of the mesh [S3, unofficial].
4. Normalize both $\mathbf{n}_i$ and $\mathbf{n}_i^*$ to unit length.
5. Compute the dot product $d_i = \mathbf{n}_i \cdot \mathbf{n}_i^*$.
6. Apply the chosen formula (NC or MCE) to aggregate the errors.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def compute_normal_consistency(estimated_normals: np.ndarray, ground_truth_normals: np.ndarray) -> float:
    """
    Computes the Normal Consistency (NC) metric.
    
    Args:
        estimated_normals: Array of shape (N, 3) with estimated unit normals.
        ground_truth_normals: Array of shape (N, 3) with ground truth unit normals.
        
    Returns:
        NC value as a percentage (0-100).
    """
    # Ensure inputs are float32/64 for precision
    estimated_normals = np.asarray(estimated_normals, dtype=np.float64)
    ground_truth_normals = np.asarray(ground_truth_normals, dtype=np.float64)
    
    # Normalize to handle potential non-unit vectors from gradient computation
    est_norms = estimated_normals / (np.linalg.norm(estimated_normals, axis=1, keepdims=True) + 1e-8)
    gt_norms = ground_truth_normals / (np.linalg.norm(ground_truth_normals, axis=1, keepdims=True) + 1e-8)
    
    # Compute dot products
    # Clamp to [-1, 1] to avoid numerical errors in acos if using MCE, 
    # though NC formula (1 - dot) is robust to slight overshoots if dot > 1.
    dots = np.sum(est_norms * gt_norms, axis=1)
    
    # Standard NC formula: 1 - cos(angle)
    # cos(angle) is the dot product for unit vectors
    errors = 1.0 - dots
    
    # Average error
    mean_error = np.mean(errors)
    
    # Convert to percentage
    nc_percentage = mean_error * 100.0
    
    return nc_percentage

def compute_mean_cosine_error(estimated_normals: np.ndarray, ground_truth_normals: np.ndarray) -> float:
    """
    Computes the Mean Cosine Error (MCE) in radians.
    
    Args:
        estimated_normals: Array of shape (N, 3).
        ground_truth_normals: Array of shape (N, 3).
        
    Returns:
        MCE value in radians.
    """
    est_norms = estimated_normals / (np.linalg.norm(estimated_normals, axis=1, keepdims=True) + 1e-8)
    gt_norms = ground_truth_normals / (np.linalg.norm(ground_truth_normals, axis=1, keepdims=True) + 1e-8)
    
    dots = np.sum(est_norms * gt_norms, axis=1)
    
    # Clamp to [-1, 1] to prevent NaN from acos due to floating point precision
    dots = np.clip(dots, -1.0, 1.0)
    
    # Compute angle in radians
    angles = np.arccos(dots)
    
    return np.mean(angles)
```

**Parameters:**
- $N$: Number of points. Typical values range from thousands to millions depending on mesh resolution [S3, unofficial].
- Batch size: When processing large meshes, points are often processed in batches (e.g., 16,384) to manage memory [S3, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Dot products may slightly exceed $[-1, 1]$ due to floating-point arithmetic. If using the MCE variant ($\arccos$), this must be clamped to $[-1, 1]$ to avoid NaNs [S3, unofficial]. The NC variant ($1 - \text{dot}$) is less sensitive but should still be monitored.
- **Normal Orientation:** Surface normals can be flipped (inward vs outward). If the ground truth and estimated normals are opposite, the dot product is $-1$, resulting in an error of $2$ (NC) or $\pi$ radians (MCE). Some implementations use the absolute value of the dot product $|\mathbf{n}_i \cdot \mathbf{n}_i^*|$ to ignore orientation ambiguity, effectively treating opposite normals as consistent [S3, unofficial][S4]. This is common when the orientation is not guaranteed to be consistent.
- **Degenerate Input:** If the mesh has no surface (empty), $N=0$, the metric is undefined.

## Subtopics

### Angular Normal Error Metric
A variant that explicitly reports the angle in degrees and provides statistical summaries (MAE, Std, Max, Median) rather than a single mean value [S3, unofficial]. It samples points on the surface (default $N=2,097,152$), computes network gradients, and compares them against mesh face normals [S3, unofficial]. It uses the formula $\theta = \arccos( | \mathbf{n}_{mesh} \cdot \mathbf{n}_{net} | ) \times \frac{180}{\pi}$ [S3, unofficial]. This variant is useful for detailed debugging of normal estimation quality [S3, unofficial].

### Evaluation Metrics for Depth and Normal Maps
Extends normal consistency to 2D image space for depth and normal map evaluation [S4]. It includes:
1. **RMSE Normal Error:** $\sqrt{\frac{1}{n} \sum (\cos^{-1}(\frac{|\mathbf{n} \cdot \mathbf{n}^*|}{||\mathbf{n}|| ||\mathbf{n}^*||}))^2}$ [S4].
2. **Prop 30 deg:** The proportion of pixels where the angle error is less than 30 degrees: $\frac{1}{n} \sum \mathbb{I}(\cos^{-1}(\frac{|\mathbf{n} \cdot \mathbf{n}^*|}{||\mathbf{n}|| ||\mathbf{n}^*||}) < 30^\circ)$ [S4].
These metrics are used when evaluating rendered outputs rather than explicit 3D meshes [S4].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of points. The computation involves a dot product and a mean aggregation per point.
- **Memory Complexity:** $O(N)$ to store the normal vectors.
- **Performance:** Computationally inexpensive. Can be batched for large meshes (e.g., batch size 16,384) to fit within GPU memory [S3, unofficial].
- **Strengths:** Directly measures geometric orientation accuracy, which is critical for applications like lighting and shading [S2]. It is robust to small positional errors if the normals are correct.
- **Limitations:** It does not measure positional accuracy (Chamfer Distance does that) [S1]. It is sensitive to normal flipping if not handled (e.g., via absolute dot product) [S3, unofficial]. It cannot represent the variance of samples as effectively as correlation metrics derived from lighting [S2].
- **When to use:** When surface orientation is more important than exact vertex positions, or as a complementary metric to Chamfer Distance in 3D reconstruction [S1].

## Background
Normal Consistency was introduced as a standard metric for 3D surface reconstruction tasks to evaluate the geometric consistency of estimated surfaces [S1]. It is frequently paired with Chamfer Distance to provide a comprehensive evaluation of both position and orientation [S1]. The Mean Cosine Error (MCE) variant was used in early volume rendering gradient estimation comparisons to measure the accuracy of gradient reconstruction [S2]. More recently, it has been adopted in neural implicit surface learning (e.g., TetraSDF, HelixSurf) to evaluate the quality of SDF gradients and mesh normals [S3, unofficial][S4].

## Sources
[S1] Self-Supervised Dual Contouring, https://doi.org/10.1109/cvpr52733.2024.00448, fetched 2026-10-26
[S2] A Comparison of Gradient Estimation Methods for Volume Rendering on Unstructured Meshes, https://www.computer.org/csdl/journal/tg/2011/03/ttg2011030305/13rRUx0xPi5, fetched 2026-11-01
[S3] naver-ai/TetraSDF, https://github.com/naver-ai/TetraSDF, fetched 2026-10-27 (unofficial)
[S4] HelixSurf: A Robust and Efficient Neural Implicit Surface Learning of Indoor Scenes with Iterative Intertwined Regularization, https://arxiv.org/abs/2302.14340v2, fetched 2026-10-26
