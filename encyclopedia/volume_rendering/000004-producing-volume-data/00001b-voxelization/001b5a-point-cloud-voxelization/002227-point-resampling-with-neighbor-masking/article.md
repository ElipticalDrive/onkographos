# Point Resampling with Neighbor Masking

Also called: Point Resampling

## Overview
Point Resampling with Neighbor Masking is a method for selectively reducing the density of a point cloud or tetrahedral grid by identifying and discarding points based on their topological relationship to active regions [S1, unofficial]. It solves the problem of maintaining geometric fidelity while compressing data, ensuring that points critical to the structure (neighbors of active elements) are preserved while redundant points are removed [S1, unofficial]. It distinguishes itself from its siblings by using explicit neighbor masks derived from Signed Distance Fields (SDF) and tetrahedral connectivity rather than random selection or error-driven metrics [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A set of points `x_nx3` (shape $N \times 3$), a Signed Distance Field `sdf`, and a tetrahedral mesh `tets` (indices of vertices forming tetrahedra).
- Output: A boolean mask `mask` of length $N$, where `True` indicates a point to be kept (resampled) and `False` indicates a point to be discarded.

**Data Structures:**
- `mask`: A boolean array of size $N$.
- `neighbor_active_points_ids`: A list or tensor of indices representing points that are neighbors to active tetrahedra.
- `active_points_ids`: A list or tensor of indices representing points that are currently active (e.g., inside or near the isosurface).

**Steps:**
The method supports three strategies: Partial, Percentage, and Full resampling.

1. **Partial Resampling:**
   - Identify neighbor points: Call `neighbor_active_points(sdf, tets)` to get `neighbor_active_points_ids`. These are points adjacent to tetrahedra that are active (intersecting or near the isosurface).
   - Create mask: Initialize `mask` of size $N$ with `False`.
   - Set `mask[neighbor_active_points_ids] = True`.
   - The number of resampled points is the count of `True` values in `mask`.

2. **Percentage Resampling:**
   - Identify active points: Call `get_active_points(sdf, tets)` to get `active_points_ids`.
   - Identify neighbor points: Call `neighbor_active_points(sdf, tets)` to get `neighbor_active_points_ids`.
   - Create mask: Initialize `mask` of size $N$ with `False`.
   - Set `mask[active_points_ids] = True`.
   - Find indices where mask is `True`: `true_indices = where(mask == True)`.
   - Add neighbors: Set `mask[neighbor_active_points_ids] = True`.
   - Update `true_indices` to reflect all `True` positions in the updated mask.
   - Shuffle `true_indices` randomly.
   - Determine the number of points to keep: $K = \lfloor 0.9 \times \text{len}(true\_indices) \rfloor$ (or another specified percentage).
   - Set the first $K$ indices in the shuffled `true_indices` to `False` in the mask.
   - The number of resampled points is the count of `True` values in `mask`.

3. **Full Resampling:**
   - Set `nb_resampled_points = x_nx3.shape[0]`.
   - Create `mask` of size $N$ with all `False`.
   - This strategy implies no points are masked out by the logic, effectively keeping all points if the mask is inverted or used as a filter for removal. *Note: The source description is ambiguous here; it sets an all-False mask but defines the count as the total points. Implementation likely inverts this mask for removal or uses it as a "keep all" flag.*

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def resample_points(x_nx3, sdf, tets, strategy='partial', percentage=0.9):
    """
    Resample points based on neighbor masking.
    
    Args:
        x_nx3: Array of shape (N, 3) containing point coordinates.
        sdf: Signed Distance Field function or grid.
        tets: Array of tetrahedron vertex indices.
        strategy: 'partial', 'percentage', or 'full'.
        percentage: Fraction of active+neighbor points to keep (for 'percentage' strategy).
        
    Returns:
        mask: Boolean array of shape (N,) where True means keep the point.
    """
    N = x_nx3.shape[0]
    mask = torch.zeros(N, dtype=torch.bool)
    
    if strategy == 'partial':
        # Get indices of points that are neighbors to active tetrahedra
        neighbor_ids = neighbor_active_points(sdf, tets)
        mask[neighbor_ids] = True
        
    elif strategy == 'percentage':
        # Get indices of active points
        active_ids = get_active_points(sdf, tets)
        # Get indices of neighbor points
        neighbor_ids = neighbor_active_points(sdf, tets)
        
        # Mark active points as True
        mask[active_ids] = True
        
        # Mark neighbor points as True
        mask[neighbor_ids] = True
        
        # Get all indices that are currently True
        true_indices = torch.where(mask)[0]
        
        # Shuffle the indices
        shuffled_indices = true_indices[torch.randperm(len(true_indices))]
        
        # Calculate how many to keep
        num_to_keep = int(len(shuffled_indices) * percentage)
        
        # Set the first num_to_keep to False (discard them)
        # The rest remain True
        mask[shuffled_indices[:num_to_keep]] = False
        
    elif strategy == 'full':
        # Keep all points (mask is False, implying no removal via this mask logic, 
        # or mask is inverted in usage. Based on source: "create all-False mask")
        mask = torch.zeros(N, dtype=torch.bool)
        
    return mask

# Helper functions assumed to be defined elsewhere in the TetWeave context
def neighbor_active_points(sdf, tets):
    # Returns indices of points adjacent to tetrahedra that are active
    pass

def get_active_points(sdf, tets):
    # Returns indices of points that are active (e.g., inside isosurface)
    pass
```

**Parameters:**
- `strategy`: String, one of `'partial'`, `'percentage'`, `'full'`. Default: `'partial'`.
- `percentage`: Float, fraction of points to keep in percentage mode. Default: `0.9` (90%).

**Edge Cases and Pitfalls:**
- **Precision:** SDF evaluation must be precise to correctly identify active vs. inactive regions.
- **Boundaries:** Points on the boundary of the domain may have fewer neighbors, affecting the mask density.
- **Degenerate Input:** If `tets` is empty or `sdf` is undefined, the helper functions will fail. [G] Ensure `tets` and `sdf` are valid before calling.

## Subtopics

### [Error-Driven Point Resampling & Grid Aggregation](5425-error-driven-point-resampling-grid-aggregation.md)
Drives resampling based on rendering error density rather than topological neighbor status.

### [Random Point Subsampling](6215-random-point-subsampling.md)
A deterministic-free downsampling primitive used within the pipeline for baseline reduction.

### [A-buffer Sampling Method](12360-a-buffer-sampling-method.md)
Maps points to a grid using weighted averaging, differing in its aggregation approach.

## Operating characteristics
- **Time Complexity:** Depends on the cost of `get_active_points` and `neighbor_active_points`, which typically involve traversing the tetrahedral mesh and evaluating the SDF. For $N$ points and $T$ tetrahedra, neighbor identification is $O(T)$ or $O(N)$ depending on implementation. Shuffling is $O(N)$.
- **Memory Complexity:** $O(N)$ for the mask and indices.
- **Performance:** The method is efficient for compression, reducing file size significantly (e.g., from 1.0MB to 345.5kB in the cited example) by discarding inactive points [S1, unofficial].
- **Strengths:** Preserves geometric structure by keeping neighbors of active regions. Simple to implement.
- **Limitations:** Relies on the quality of the SDF and tetrahedral mesh. May not preserve fine details if the percentage is too low.
- **When to use:** When compressing point clouds or tetrahedral grids for storage or transmission while maintaining structural integrity.
- **When not to use:** When uniform random sampling is sufficient or when error-driven refinement is required.

## Background
The method originates from the TetWeave project, which focuses on isosurface extraction using on-the-fly Delaunay tetrahedral grids for gradient-based mesh optimization [S1, unofficial]. It was developed by Alexandre Binninger, Ruben Wiersma, Philipp Herholz, and Olga Sorkine-Hornung for SIGGRAPH 2025 [S1, unofficial]. The technique is part of a pipeline that includes compression via discarding inactive points and converting values to `float16` [S1, unofficial].

## Sources
[S1] AlexandreBinninger/TetWeave, https://github.com/AlexandreBinninger/TetWeave, fetched 2026-10-27 (unofficial)
