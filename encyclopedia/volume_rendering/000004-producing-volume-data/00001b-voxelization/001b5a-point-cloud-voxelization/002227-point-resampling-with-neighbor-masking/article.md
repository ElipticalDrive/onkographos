# Point Resampling with Neighbor Masking

Also called: Point Resampling

## Overview
Point Resampling with Neighbor Masking resamples a point cloud based on active and neighbor status to support partial, percentage, and full resampling strategies [S1, unofficial]. It controls point density by selectively activating points tied to tetrahedral connectivity [S1, unofficial].

## How it works
The method operates on a set of points $X \in \mathbb{R}^{N \times 3}$ and a set of tetrahedra $T$. It produces a boolean mask $M$ of length $N$, where $M[i] = \text{True}$ indicates the point $i$ is active (kept) and $\text{False}$ indicates it is inactive (discarded or compressed) [S1, unofficial].

### Inputs and Outputs
- **Inputs**:
  - `x_nx3`: Tensor of shape $(N, 3)$ containing point coordinates.
  - `tets`: Tensor of shape $(K, 4)$ containing tetrahedron vertex indices.
  - `sdf`: Tensor of shape $(N)$ or $(K)$ containing signed distance values (used to determine "active" status) [S1, unofficial].
  - `strategy`: String, one of `"partial"`, `"percentage"`, or `"full"` [S1, unofficial].
- **Outputs**:
  - `mask`: Boolean tensor of shape $(N)$ [S1, unofficial].
  - `nb_resampled_points`: Integer count of active points [S1, unofficial].

### Data Structures
- **Neighbor Map**: A mapping from each tetrahedron to its constituent vertices, and vice versa. The function `neighbor_active_points(sdf, tets)` identifies vertices that are part of tetrahedra where the SDF indicates activity (e.g., near the isosurface) [S1, unofficial].
- **Active Points Set**: The set of indices $I_{active}$ where the SDF value meets a specific criterion (e.g., $|SDF| < \epsilon$) [S1, unofficial].

### Steps
1. **Identify Active Points**: Determine the set of indices $I_{active}$ corresponding to points considered "active" based on the SDF and tetrahedron connectivity [S1, unofficial].
2. **Identify Neighbor Active Points**: Determine the set of indices $I_{neighbor}$ corresponding to points that are neighbors to the active tetrahedra [S1, unofficial].
3. **Apply Strategy**: Construct the mask $M$ based on the chosen strategy:
   - **Partial**: Activate only the neighbor active points [S1, unofficial].
   - **Percentage**: Activate all active points, plus a subset of neighbor points to maintain a specific ratio or percentage of the total [S1, unofficial].
   - **Full**: Activate all points [S1, unofficial].
4. **Count**: Calculate the number of resampled points as the sum of `True` values in $M$ [S1, unofficial].

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
    N = len(x_nx3)
    mask = [False] * N
    
    if strategy == "partial":
        # Partial method: get neighbor_active_points_ids via neighbor_active_points(sdf, tets)
        neighbor_ids = neighbor_active_points(sdf, tets)
        for idx in neighbor_ids:
            mask[idx] = True
        # count resampled points as sum of ~mask (interpreted as sum of active/True values)
        nb_resampled_points = sum(mask)
        
    elif strategy == "percentage":
        # Percentage method: get active_points_ids via get_active_points(sdf, tets)
        active_ids = get_active_points(sdf, tets)
        neighbor_ids = neighbor_active_points(sdf, tets)
        
        # create mask with True at active indices
        for idx in active_ids:
            mask[idx] = True
            
        # get true_indices where mask is True
        true_indices = [i for i, v in enumerate(mask) if v]
        
        # set neighbor indices to True
        for idx in neighbor_ids:
            mask[idx] = True
            
        # get true_indices where mask is True (active + neighbors)
        true_indices = [i for i, v in enumerate(mask) if v]
        
        # shuffle true_indices
        import random
        random.shuffle(true_indices)
        
        # flip first 90% of shuffled indices to False
        flip_count = int(0.9 * len(true_indices))
        for idx in true_indices[:flip_count]:
            mask[idx] = False
            
        # count resampled points as sum of ~mask (interpreted as sum of active/True values)
        nb_resampled_points = sum(mask)
        
    elif strategy == "full":
        # Full method: set nb_resampled_points to x_nx3.shape[0]
        nb_resampled_points = N
        # create all-False mask
        mask = [False] * N
        
    else:
        raise ValueError(f"Unknown strategy: {strategy}")
        
    return mask, nb_resampled_points

def get_active_points(sdf, tets):
    # Returns unique vertex indices from tetrahedra meeting activity criteria
    # Not given in the sources. [G]
    active_tets = [i for i, v in enumerate(sdf) if abs(v) < 0.1]
    vertices = set()
    for t in active_tets:
        vertices.update(tets[t])
    return sorted(vertices)

def neighbor_active_points(sdf, tets):
    # Returns vertices that are neighbors to active tetrahedra
    # Not given in the sources. [G]
    # Assumes it returns the union of active vertices and their immediate mesh neighbors
    return get_active_points(sdf, tets)
```

### Parameters
- **Strategy**: `"partial"`, `"percentage"`, `"full"` [S1, unofficial].
- **Percentage Threshold**: For the `"percentage"` strategy, the source specifies flipping the first 90% of shuffled indices to False, leaving 10% active [S1, unofficial].

### Edge Cases and Pitfalls
- **Precision**: The source mentions converting values to `float16` for compression, which may introduce precision loss [S1, unofficial].
- **Boundaries**: The definition of "neighbor" must be consistent with the tetrahedral mesh connectivity [S1, unofficial].
- **Degenerate Input**: If no active tetrahedra exist, the mask will be all False for partial/percentage strategies [S1, unofficial].

## Subtopics

### [Error-Driven Point Resampling & Grid Aggregation](5425-error-driven-point-resampling-grid-aggregation.md)
This method computes a 3D error density field by accumulating per-pixel rendering discrepancies across multiple camera views into a voxel grid, then uses this density to guide the resampling of points for tetrahedral grid refinement. It distinguishes itself from its parent [Point Resampling with Neighbor Masking](2710-point-resampling-with-neighbor-masking.md) by driving the resampling probability based on quantitative rendering error rather than topological neighbor status.

### Random Point Subsampling
Selects a random subset of points from a given set without replacement [S2, unofficial]. It generates a random permutation of indices, indexes the points tensor to shuffle them, reshapes to 2D, and returns the subset [S2, unofficial]. Parameters are the input points tensor (typically shape (M, 3)) and the number of points to select (n_points) [S2, unofficial].

## Operating characteristics
- **Time Complexity**: Depends on the neighbor lookup and sorting/shuffling operations. The source does not provide explicit Big-O notation [S1, unofficial].
- **Memory Complexity**: The mask requires $O(N)$ bits. The source notes that discarding inactive points and converting to `float16` can compress the model from 1.0MB to 345.5kB for a sample model [S1, unofficial].
- **Strengths**: Allows for controlled reduction of point cloud density, which can speed up optimization and reduce memory usage [S1, unofficial].
- **Limitations**: The fixed 90% flip in the percentage strategy may not be optimal for all scenes [S1, unofficial].
- **When to use**: When computational resources are limited and a simplified point cloud representation is acceptable [S1, unofficial].
- **When not to use**: When high-fidelity reconstruction is required, or when the active region is sparse and neighbor masking might remove critical details [S1, unofficial].

## Background
The method is part of the TetWeave framework, which performs isosurface extraction using on-the-fly Delaunay tetrahedral grids for gradient-based mesh optimization [S1, unofficial]. It was introduced by Alexandre Binninger, Ruben Wiersma, Philipp Herholz, and Olga Sorkine-Hornung in their SIGGRAPH 2025 paper "TetWeave: Isosurface Extraction using On-The-Fly Delaunay Tetrahedral Grids for Gradient-Based Mesh Optimization" [S1, unofficial].

## Sources
[S1] AlexandreBinninger/TetWeave, https://github.com/AlexandreBinninger/TetWeave, fetched 2026-10-27 (unofficial)
[S2] omkarchittar/Neural_Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
