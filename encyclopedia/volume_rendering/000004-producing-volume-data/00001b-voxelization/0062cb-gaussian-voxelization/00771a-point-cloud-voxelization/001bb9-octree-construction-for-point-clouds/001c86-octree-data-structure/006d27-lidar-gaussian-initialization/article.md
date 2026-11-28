# LiDAR-Gaussian Initialization

Also called: Initialization of Gaussians with LiDAR Measurement

## Overview
This method initializes 3D Gaussian splats from LiDAR point cloud data to serve as the radiance field representation for mapping and rendering. It solves the problem of establishing geometrically accurate initial parameters (position, covariance, and orientation) for Gaussians directly from sparse LiDAR measurements, leveraging the planar characteristics of the scanned environment. It differs from sibling methods by explicitly using size-adaptive voxel subdivision based on surface thickness indicators ($\eta$) and deriving Gaussian covariances from the local point distribution's eigenstructure, rather than using fixed-size voxels or simple nearest-neighbor averaging.

## How it works

### Inputs and Outputs
*   **Input:** A LiDAR point cloud $\{w_{pi}\}_{i=1}^{N}$, where $w_{pi}$ is the 3D position of the $i$-th point in the world frame.
*   **Output:** A set of 3D Gaussians, each defined by a center $w_{pi}$, a covariance matrix $\Sigma_{w_{pi}}$, and a scaling factor $\alpha_i$.

### Data Structures
The method operates on a hierarchical voxel structure (implied by the "size-adaptive" and "subdivision" steps). Each voxel node contains:
*   `points`: List of 3D positions $w_{pi}$ belonging to the voxel.
*   `mean`: 3D vector $p$.
*   `covariance`: $3 \times 3$ matrix $\Sigma_{n,p}$.
*   `eigenvalues`: Array $[\lambda_{min}, \lambda_{mid}, \lambda_{max}]$.
*   `eigenvector`: 3D vector $n$ (normal).
*   `is_leaf`: Boolean flag indicating if the voxel should be subdivided.

### Steps

1.  **Voxel Partitioning:** Partition the LiDAR point cloud into size-adaptive voxels. The initial voxel size is determined by the input resolution or a predefined grid.
2.  **Statistical Analysis:** For each voxel containing $N$ points:
    a. Compute the mean position $p$ [S1]:
    $$ p = \frac{1}{N} \sum_{i=1}^{N} w_{pi} $$
    b. Compute the covariance matrix $\Sigma_{n,p}$ [S1]:
    $$ \Sigma_{n,p} = \frac{1}{N} \sum_{i=1}^{N} (w_{pi} - p)(w_{pi} - p)^T $$
    c. Compute the eigenvalues $\lambda_{min}, \lambda_{mid}, \lambda_{max}$ and the corresponding eigenvector $n$ of $\Sigma_{n,p}$, sorted such that $\lambda_{min} \le \lambda_{mid} \le \lambda_{max}$ [S1]. The eigenvector $n$ associated with $\lambda_{min}$ is the normal vector of the planar surface [S1].
3.  **Thickness Evaluation:** Calculate the planar surface thickness indicator $\eta$ [S1]:
    $$ \eta = \frac{\lambda_{min}}{\sqrt{\lambda_{mid}^2 + \lambda_{min}^2 + \lambda_{max}^2}} $$
4.  **Subdivision Decision:** If $\eta$ indicates significant thickness (i.e., the surface is not sufficiently planar or thin), subdivide the voxel into finer levels and repeat steps 2–4 for the child voxels. If the voxel is sufficiently divided, proceed to initialization [S1].
5.  **Scaling Factor Calculation:** Determine a scaling factor $\alpha_i$ for each point $w_{pi}$ based on the local point density within the voxel [S1]. [G: Typically, $\alpha_i$ is inversely proportional to the number of points in the voxel or the local density estimate to prevent over-sampling in dense regions.]
6.  **Covariance Rescaling:** Rescale the planar Gaussian covariance for each point [S1]:
    $$ \Sigma_{w_{pi}} = \alpha_i \Sigma_{n,p} $$
7.  **Gaussian Definition:** Define the 3D Gaussian for the $i$-th point [S1]:
    $$ G^3D_i(w_p) = \exp\left(- (w_p - w_{pi})^T \Sigma_{w_{pi}}^{-1} (w_p - w_{pi})\right) $$
    where $w_p$ is the position in the world frame where the Gaussian is evaluated.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_covariance(points):
    """Compute mean and covariance matrix from a list of 3D points."""
    N = len(points)
    if N == 0:
        return None, None
    
    points_arr = np.array(points)
    mean = np.mean(points_arr, axis=0)
    centered = points_arr - mean
    cov = (centered.T @ centered) / N
    return mean, cov

def get_eigen_info(cov):
    """Compute eigenvalues and eigenvectors, sorted by magnitude."""
    eigenvalues, eigenvectors = np.linalg.eigh(cov)
    # Sort by eigenvalue magnitude (ascending)
    sorted_indices = np.argsort(eigenvalues)
    sorted_eigenvalues = eigenvalues[sorted_indices]
    sorted_eigenvectors = eigenvectors[:, sorted_indices]
    return sorted_eigenvalues, sorted_eigenvectors

def compute_eta(eigenvalues):
    """Compute planar surface thickness indicator."""
    lam_min, lam_mid, lam_max = eigenvalues
    denominator = np.sqrt(lam_mid**2 + lam_min**2 + lam_max**2)
    if denominator == 0:
        return 0.0
    return lam_min / denominator

def initialize_gaussians_from_lidar(point_cloud, initial_voxel_size, eta_threshold):
    """
    Initialize Gaussians from LiDAR point cloud using size-adaptive voxels.
    
    Args:
        point_cloud: List of 3D points (numpy arrays).
        initial_voxel_size: Size of the initial voxel grid.
        eta_threshold: Threshold for eta to decide subdivision. 
                       Note: Source [S1] says "if eta indicates significant thickness, subdivide".
                       This implies a high eta (large lambda_min relative to others) means thick/non-planar.
    
    Returns:
        List of Gaussian parameters: (center, covariance, alpha)
    """
    gaussians = []
    
    # Helper to process a voxel
    def process_voxel(points, current_voxel_size):
        # 1. Statistical Analysis
        mean, cov = compute_covariance(points)
        if mean is None:
            return
        
        eigenvalues, eigenvectors = get_eigen_info(cov)
        lam_min, lam_mid, lam_max = eigenvalues
        normal = eigenvectors[:, 0] # Eigenvector for lambda_min
        
        # 2. Thickness Evaluation
        eta = compute_eta(eigenvalues)
        
        # 3. Subdivision Decision
        # If eta is large (significant thickness), subdivide
        if eta > eta_threshold and current_voxel_size > some_min_size:
            # Subdivide voxel into 8 children
            child_points = partition_points_into_children(points, mean, current_voxel_size)
            child_size = current_voxel_size / 2.0
            for child_pts in child_points:
                process_voxel(child_pts, child_size)
        else:
            # 4. Initialization
            # 5. Scaling Factor
            # [G] Density-based scaling. Inverse of point count is a common proxy.
            N = len(points)
            alpha = 1.0 / N if N > 0 else 1.0
            
            # 6. Covariance Rescaling
            cov_rescaled = alpha * cov
            
            # 7. Gaussian Definition
            # Store center, covariance, and alpha
            for i, w_pi in enumerate(points):
                # Each point becomes a Gaussian? Or one Gaussian per voxel?
                # Source [S1] says "for each point... rescale... G^3D_i".
                # This implies one Gaussian per point, initialized with the voxel's stats.
                gaussians.append({
                    'center': w_pi,
                    'covariance': cov_rescaled,
                    'alpha': alpha,
                    'normal': normal
                })

    # Initial partition
    # [G] Assuming a simple grid partition for the initial step
    initial_voxels = partition_point_cloud(point_cloud, initial_voxel_size)
    for voxel_points in initial_voxels:
        process_voxel(voxel_points, initial_voxel_size)
        
    return gaussians
```

### Parameters
*   $N$: Number of points within the voxel [S1].
*   $w_{pi}$: Position of the $i$-th point within the voxel [S1].
*   $\Sigma_{n,p}$: Covariance matrix of the voxel [S1].
*   $\eta$: Indicator for planar surface thickness [S1].
*   $\lambda_{min}, \lambda_{mid}, \lambda_{max}$: Eigenvalues of $\Sigma_{n,p}$ sorted by magnitude [S1].
*   $\alpha_i$: Scaling factor for the $i$-th point, determined by point density within the voxel [S1].
*   $G^3D_i(w_p)$: 3D Gaussian value at position $w_p$ for the $i$-th Gaussian [S1].
*   $w_p$: Position in world frame where Gaussian is evaluated [S1].

### Edge Cases and Pitfalls
*   **Zero Points:** If a voxel contains no points, covariance and eigenvalues are undefined. The algorithm must skip or handle empty voxels.
*   **Degenerate Covariance:** If all points in a voxel are identical, $\Sigma_{n,p}$ is zero, leading to division by zero in $\eta$ or inversion in the Gaussian. [G: Add a small epsilon to eigenvalues or skip such voxels.]
*   **Threshold Selection:** The threshold for $\eta$ determines the granularity of the map. A too-low threshold leads to excessive subdivision and high memory usage; a too-high threshold results in poor planar approximation.

## Operating characteristics
*   **Time Complexity:** Depends on the number of points $M$ and the depth of the octree. Each point is processed at each level of subdivision. In the worst case (deep subdivision), it is $O(M \cdot D)$ where $D$ is the depth.
*   **Memory Complexity:** Proportional to the number of Gaussians generated, which is equal to the number of LiDAR points $M$ (one Gaussian per point) plus the octree structure overhead.
*   **Strengths:** Provides geometrically accurate initial normals and covariances based on local surface structure. Adaptive voxel size ensures detail where needed.
*   **Limitations:** Computationally intensive due to recursive subdivision and eigen-decomposition for each voxel. Sensitive to noise in LiDAR data, which can affect $\eta$ and normal estimation.

## Background
This method originates from the LIV-GaussMap paper, which introduces LiDAR-Inertial-Visual fusion for real-time 3D radiance field map rendering. It builds on the octree approach discussed in [12] (likely referring to a prior work on octree-based LiDAR mapping) and adapts it for Gaussian Splatting initialization. The use of eigenvalues to determine surface planarity is a standard technique in point cloud processing, adapted here for radiance field initialization.

## Sources
[S1] LIV-GaussMap: LiDAR-Inertial-Visual Fusion for Real-Time 3D Radiance Field Map Rendering, https://doi.org/10.1109/lra.2024.3400149, fetched 2026-10-26
