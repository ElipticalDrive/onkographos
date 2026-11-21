# Dynamic Voxelization (Coordinate and Feature)

Also called: Dynamic Voxelization with Scatter Reduction

## Overview
Dynamic Voxelization (Coordinate and Feature) converts point clouds into variable-size voxel grids by simultaneously computing voxel coordinates and aggregating pointwise features into those voxels [S1, unofficial]. It differs from [Dynamic Voxelization (Coordinate Only)](3347-dynamic-voxelization-coordinate-only.md) by performing scatter-reduction on features during the voxelization process, rather than returning only coordinate mappings. It addresses the inefficiency of hard voxelization when handling sparse, variable-density point clouds by avoiding fixed grid allocation and duplicate handling overhead [S1, unofficial].

## How it works
The method decomposes voxelization into three sequential stages: coordinate calculation, indexing, and scatter-reduction [S1, unofficial].

### Inputs and Outputs
- **Input**: Point cloud tensor $P \in \mathbb{R}^{N \times D}$, where $N$ is the number of points and $D$ is the feature dimension (typically $D=4$ for $(x, y, z, \text{reflectivity})$ or more).
- **Input**: Voxelization parameters: voxel size $v \in \mathbb{R}^3$, point cloud range $R \in \mathbb{R}^6$ (min/max $x, y, z$), and max points per voxel $M$ (often ignored in dynamic mode or set to 1 for aggregation).
- **Output**: Voxel features tensor $V \in \mathbb{R}^{K \times F}$, where $K$ is the number of non-empty voxels and $F$ is the feature dimension.
- **Output**: Voxel coordinates tensor $C \in \mathbb{R}^{K \times 3}$, representing the grid indices of the non-empty voxels.

### Data Structures
- **Point Cloud**: A flat tensor of shape $(N, D)$.
- **Voxel Coordinates**: A tensor of shape $(K, 3)$ storing integer grid indices $(i, j, k)$ for each non-empty voxel.
- **Routing Index**: A 1D tensor of shape $(N,)$ mapping each point to its target voxel index in the output feature tensor.

### Steps
1. **Calculate Target Coordinates**: For each point $p_n = (x_n, y_n, z_n, \dots)$, compute its voxel grid coordinates $(i_n, j_n, k_n)$ using the voxel size $v$ and range $R$:
   $$
   i_n = \left\lfloor \frac{x_n - R_{min,x}}{v_x} \right\rfloor, \quad
   j_n = \left\lfloor \frac{y_n - R_{min,y}}{v_y} \right\rfloor, \quad
   k_n = \left\lfloor \frac{z_n - R_{min,z}}{v_z} \right\rfloor
   $$
   Clamp coordinates to valid range $[0, K_{max}-1]$ where $K_{max}$ is derived from the range and voxel size.

2. **Indexing (Unique and Routing)**:
   - Collect all computed coordinates $(i_n, j_n, k_n)$ for $n=0 \dots N-1$.
   - Use `torch.unique` (or equivalent) on the coordinate tensor to identify unique voxel indices. This returns:
     - `unique_coors`: The set of unique voxel coordinates $(K \times 3)$.
     - `inverse_indices`: A mapping from each point's original index to its unique voxel index in `unique_coors` (shape $N$). This serves as the routing index for scatter-reduction.
   - The number of non-empty voxels $K$ is the number of unique coordinates.

3. **Scatter-Reduction**:
   - Initialize an output feature tensor $V$ of shape $(K, F)$ with zeros.
   - For each point $n$, scatter its feature vector $f_n$ into $V$ at row `inverse_indices[n]` using a reduction operation (e.g., sum, mean).
   - The reduction operation aggregates all points falling into the same voxel.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def dynamic_voxelization_and_feature(
    points: torch.Tensor,
    voxel_size: torch.Tensor,
    point_cloud_range: torch.Tensor,
    max_points_per_voxel: int = 1,  # Ignored in dynamic mode for aggregation
    max_voxels: int = 0,           # Ignored in dynamic mode
    reduce: str = "sum"
) -> tuple[torch.Tensor, torch.Tensor]:
    """
    points: (N, D) tensor, D >= 3
    voxel_size: (3,) tensor [vx, vy, vz]
    point_cloud_range: (6,) tensor [x_min, y_min, z_min, x_max, y_max, z_max]
    reduce: "sum" or "mean"
    Returns:
        coors: (K, 3) tensor of unique voxel coordinates
        features: (K, D) tensor of aggregated voxel features
    """
    # Step 1: Calculate target coordinates
    # Shift points to origin relative to min range
    shifted_points = points[:, :3] - point_cloud_range[:3]
    # Divide by voxel size to get grid indices
    coors_float = shifted_points / voxel_size
    # Floor to get integer indices
    coors_int = torch.floor(coors_float).int()
    
    # Clamp to valid range (optional but recommended for stability)
    max_coors = ((point_cloud_range[3:] - point_cloud_range[:3]) / voxel_size).int()
    coors_int = torch.clamp(coors_int, 0, max_coors - 1)
    
    # Step 2: Indexing using unique
    # Flatten coordinates for unique operation if needed, or pass as is
    # torch.unique returns sorted unique values and inverse indices
    # coors_unique: (K, 3)
    # inverse_indices: (N,) mapping each point to its row in coors_unique
    coors_unique, inverse_indices = torch.unique(
        coors_int, 
        dim=0, 
        sorted=True, 
        return_inverse=True
    )
    
    K = coors_unique.shape[0]
    N = points.shape[0]
    D = points.shape[1]
    
    # Step 3: Scatter-Reduction
    # Initialize output features
    features = torch.zeros((K, D), dtype=points.dtype, device=points.device)
    
    # Scatter point features into voxel features
    # Use scatter_add_ for sum reduction
    features.scatter_add_(0, inverse_indices.unsqueeze(1).expand(-1, D), points)
    
    if reduce == "mean":
        # Calculate count per voxel
        ones = torch.ones((N,), dtype=points.dtype, device=points.device)
        counts = torch.zeros((K,), dtype=points.dtype, device=points.device)
        counts.scatter_add_(0, inverse_indices, ones)
        # Avoid division by zero
        counts = torch.clamp(counts, min=1)
        # Divide features by counts
        features = features / counts.unsqueeze(1)
        
    return coors_unique, features
```

### Parameters
- **Voxel Size**: Typically $0.05$ to $0.1$ meters for LiDAR.
- **Point Cloud Range**: Defined by the sensor's field of view, e.g., $[-40, -40, -3, 40, 40, 1]$ meters.
- **Reduction Operation**: "sum" is default; "mean" is common for normalization.

### Edge Cases and Pitfalls
- **Extreme Density**: When points are extremely dense, the number of collisions increases, potentially causing numerical instability or reduced efficiency due to large scatter operations [S1, unofficial].
- **Redundant Indexing**: Repeated scatter-reduces for the same point cloud require redundant indexing if not cached [S1, unofficial].
- **Precision**: Floating-point division in coordinate calculation may introduce small errors; clamping is essential to prevent out-of-bounds indices.

## Subtopics

### Dynamic Voxelization (Coordinate Only)
Computes only voxel coordinates without feature aggregation. Use when max_points=-1 or when feature processing is handled separately. [Link](3347-dynamic-voxelization-coordinate-only.md)

### Hard Voxelization
Assigns points to a fixed grid with max points per voxel. Use when deterministic output size is required or for hardware acceleration. [Link](3346-hard-voxelization.md)

## Operating characteristics
- **Time Complexity**: Dominated by the indexing step (`torch.unique`), which is $O(N \log N)$ or $O(N)$ depending on implementation. Scatter-reduction is $O(N)$ [S1, unofficial].
- **Memory Complexity**: $O(K + N)$, where $K$ is the number of non-empty voxels. No fixed grid allocation, so memory scales with sparsity.
- **Performance**: Faster than hard voxelization for sparse point clouds due to no duplicate handling overhead. However, `torch.unique` can be a bottleneck for very large $N$ [S1, unofficial].
- **Strengths**: Handles variable-density point clouds efficiently; no fixed grid constraints; suitable for dynamic scenes.
- **Limitations**: Non-deterministic output order (due to `unique`); unstable in extremely dense regions; requires GPU for efficient scatter operations.
- **When to Use**: Sparse LiDAR point clouds, dynamic environments, when memory efficiency is critical.
- **When Not to Use**: Extremely dense point clouds, when deterministic output size is required, or on hardware without efficient scatter support.

## Background
Dynamic Voxelization was introduced as an alternative to hard voxelization in 3D object detection pipelines, particularly for LiDAR-based perception [S1, unofficial]. It was developed by the MMDetection3D team to address the inefficiencies of hard voxelization in handling sparse data. The method leverages PyTorch's `scatter` operations for efficient feature aggregation. It is widely used in models like PointPillars and SECOND for 3D object detection [S1, unofficial].

## Sources
[S1] [Potential Performance Improvement] Hard voxelization could have a much faster implementation (at the cost of determinism)., https://github.com/open-mmlab/mmdetection3d/issues/894, fetched 2026-10-26 (unofficial)
