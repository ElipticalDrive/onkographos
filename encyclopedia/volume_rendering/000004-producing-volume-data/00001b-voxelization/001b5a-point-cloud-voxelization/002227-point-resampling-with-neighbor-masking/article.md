# Point Resampling with Neighbor Masking

Also called: Point Resampling

## Overview
Point Resampling with Neighbor Masking is a strategy for managing point cloud density during the voxelization and optimization phases of volume rendering, specifically within the context of tetrahedral grid generation. It selectively activates or deactivates points based on their topological relationship to active elements (such as tetrahedra) and a user-defined resampling strategy. It distinguishes itself from its siblings by using explicit neighbor masks to control computational load and memory usage, rather than relying on error-driven metrics [S1, unofficial] or structural hierarchy like octrees [1070-octree-construction-for-point-clouds.md].

## How it works
The method operates on a set of points $X \in \mathbb{R}^{N \times 3}$ and a set of tetrahedra $T$. It produces a boolean mask $M$ of length $N$, where $M[i] = \text{True}$ indicates the point $i$ is active (kept) and $\text{False}$ indicates it is inactive (discarded or compressed).

### Inputs and Outputs
- **Inputs**:
  - `x_nx3`: Tensor of shape $(N, 3)$ containing point coordinates.
  - `tets`: Tensor of shape $(K, 4)$ containing tetrahedron vertex indices.
  - `sdf`: Tensor of shape $(N)$ or $(K)$ containing signed distance values (used to determine "active" status).
  - `strategy`: String, one of `"partial"`, `"percentage"`, or `"full"`.
- **Outputs**:
  - `mask`: Boolean tensor of shape $(N)$.
  - `nb_resampled_points`: Integer count of active points.

### Data Structures
- **Neighbor Map**: A mapping from each tetrahedron to its constituent vertices, and vice versa. The function `neighbor_active_points(sdf, tets)` identifies vertices that are part of tetrahedra where the SDF indicates activity (e.g., near the isosurface).
- **Active Points Set**: The set of indices $I_{active}$ where the SDF value meets a specific criterion (e.g., $|SDF| < \epsilon$).

### Steps
1. **Identify Active Points**: Determine the set of indices $I_{active}$ corresponding to points considered "active" based on the SDF and tetrahedron connectivity.
2. **Identify Neighbor Active Points**: Determine the set of indices $I_{neighbor}$ corresponding to points that are neighbors to the active tetrahedra.
3. **Apply Strategy**: Construct the mask $M$ based on the chosen strategy:
   - **Partial**: Activate only the neighbor active points.
   - **Percentage**: Activate all active points, plus a subset of neighbor points to maintain a specific ratio or percentage of the total.
   - **Full**: Activate all points.
4. **Count**: Calculate the number of resampled points as the sum of `True` values in $M$.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def point_resampling_with_neighbor_masking(x_nx3, tets, sdf, strategy="partial"):
    """
    Resamples points based on active/neighbor status.
    
    Args:
        x_nx3: Tensor (N, 3) point coordinates.
        tets: Tensor (K, 4) tetrahedron vertex indices.
        sdf: Tensor (N) or (K) signed distance values.
        strategy: 'partial', 'percentage', or 'full'.
        
    Returns:
        mask: Boolean tensor (N)
        nb_resampled_points: int
    """
    N = x_nx3.shape[0]
    mask = torch.zeros(N, dtype=torch.bool)
    
    # Helper to determine active tetrahedra based on SDF
    # Assuming SDF is defined per tetrahedron or interpolated. 
    # For this implementation, we assume a function that returns active tet indices.
    active_tet_indices = get_active_tets(sdf, tets)
    
    # Get neighbors of active tetrahedra
    # This function returns indices of vertices connected to active_tet_indices
    neighbor_active_points_ids = neighbor_active_points(sdf, tets)
    
    if strategy == "partial":
        # Activate only the neighbor active points
        mask[neighbor_active_points_ids] = True
        
    elif strategy == "percentage":
        # Get all active points (vertices of active tets)
        active_points_ids = get_active_points(sdf, tets)
        
        # Create mask with True at active indices
        mask[active_points_ids] = True
        
        # Get indices where mask is currently True
        true_indices = torch.where(mask)[0]
        
        # Set neighbor indices to True
        mask[neighbor_active_points_ids] = True
        
        # Get all currently true indices again (active + neighbors)
        true_indices = torch.where(mask)[0]
        
        # Shuffle true_indices
        shuffled_indices = true_indices[torch.randperm(len(true_indices))]
        
        # Flip first 90% of shuffled indices to False
        # This effectively keeps 10% of the active+neighbor points
        # Note: The source says "flip first 90% ... to False", implying 10% remain True.
        # However, the context of "percentage" usually implies a specific ratio.
        # We follow the source literally: flip first 90% to False.
        num_to_flip = int(0.9 * len(shuffled_indices))
        indices_to_false = shuffled_indices[:num_to_flip]
        mask[indices_to_false] = False
        
    elif strategy == "full":
        # Activate all points
        mask[:] = True
        
    else:
        raise ValueError(f"Unknown strategy: {strategy}")
        
    nb_resampled_points = mask.sum().item()
    
    return mask, nb_resampled_points

def get_active_tets(sdf, tets):
    # Placeholder for specific SDF thresholding logic
    # Not given in the sources. [G]
    # Assuming SDF is per-tetrahedron for this helper
    return torch.where(torch.abs(sdf) < 0.1)[0]

def get_active_points(sdf, tets):
    # Returns unique vertex indices from active_tets
    active_tets = get_active_tets(sdf, tets)
    active_tets_vertices = tets[active_tets]
    unique_vertices = torch.unique(active_tets_vertices)
    return unique_vertices

def neighbor_active_points(sdf, tets):
    # Returns vertices that are neighbors to active tetrahedra
    # This includes vertices of active tets and their immediate neighbors
    active_tets = get_active_tets(sdf, tets)
    active_tets_vertices = tets[active_tets]
    unique_active_vertices = torch.unique(active_tets_vertices)
    
    # Get all vertices connected to active tets
    # This is a simplification; in practice, this might involve a neighbor lookup
    # Not given in the sources. [G]
    # Assuming this function returns the union of active vertices and their neighbors
    return unique_active_vertices
```

### Parameters
- **Strategy**: `"partial"`, `"percentage"`, `"full"`. Default is not specified, but `"partial"` is described first.
- **Percentage Threshold**: For the `"percentage"` strategy, the source specifies flipping the first 90% of shuffled indices to False, leaving 10% active. This is a fixed value in the source [S1, unofficial].

### Edge Cases and Pitfalls
- **Precision**: The source mentions converting values to `float16` for compression, which may introduce precision loss [S1, unofficial].
- **Boundaries**: The definition of "neighbor" must be consistent with the tetrahedral mesh connectivity.
- **Degenerate Input**: If no active tetrahedra exist, the mask will be all False for partial/percentage strategies.

## Subtopics

### [Error-Driven Point Resampling & Grid Aggregation](5425-error-driven-point-resampling-grid-aggregation.md)
This method computes a 3D error density field by accumulating per-pixel rendering discrepancies across multiple camera views into a voxel grid, then uses this density to guide the resampling of points for tetrahedral grid refinement. It distinguishes itself from its parent [Point Resampling with Neighbor Masking](2710-point-resampling-with-neighbor-masking.md) by driving the resampling probability based on quantitative rendering error rather than topological neighbor status.

## Operating characteristics
- **Time Complexity**: Depends on the neighbor lookup and sorting/shuffling operations. The source does not provide explicit Big-O notation.
- **Memory Complexity**: The mask requires $O(N)$ bits. The source notes that discarding inactive points and converting to `float16` can compress the model from 1.0MB to 345.5kB for a sample model [S1, unofficial].
- **Strengths**: Allows for controlled reduction of point cloud density, which can speed up optimization and reduce memory usage.
- **Limitations**: The quality of the resulting mesh depends heavily on the definition of "active" and "neighbor" points. The fixed 90% flip in the percentage strategy may not be optimal for all scenes.
- **When to use**: When computational resources are limited and a simplified point cloud representation is acceptable.
- **When not to use**: When high-fidelity reconstruction is required, or when the active region is sparse and neighbor masking might remove critical details.

## Background
The method is part of the TetWeave framework, which performs isosurface extraction using on-the-fly Delaunay tetrahedral grids for gradient-based mesh optimization. It was introduced by Alexandre Binninger, Ruben Wiersma, Philipp Herholz, and Olga Sorkine-Hornung in their SIGGRAPH 2025 paper "TetWeave: Isosurface Extraction using On-The-Fly Delaunay Tetrahedral Grids for Gradient-Based Mesh Optimization" [S1, unofficial]. The codebase is available on GitHub [S1, unofficial].

## Sources
[S1] AlexandreBinninger/TetWeave, https://github.com/AlexandreBinninger/TetWeave, fetched 2026-10-27 (unofficial)
