# Random Point Subsampling

Also called: Random Subset Selection, Shuffle and Slice

## Overview
Random Point Subsampling selects a fixed-size random subset of points from an input set without replacement [S1, unofficial]. It serves as a deterministic-free downsampling primitive within the Point Resampling with Neighbor Masking pipeline, providing a baseline for reducing point cloud density [S1, unofficial]. Unlike Error-Driven Point Resampling & Grid Aggregation, which prioritizes geometric fidelity or grid-based density, this method ignores spatial distribution, treating all points as equally probable candidates [S1, unofficial].

## How it works
**Inputs:**
- `points`: A tensor of shape $(M, 3)$ containing $M$ 3D points [S1, unofficial].
- `n_points`: The integer count of points to select [S1, unofficial].

**Outputs:**
- A tensor of shape $(N, 3)$ where $N = \min(M, \text{n\_points})$ [S1, unofficial].

**Data Structures:**
- The input `points` tensor is stored in contiguous memory. No auxiliary structures are required beyond the permutation index array [S1, unofficial].

**Steps:**
1. Determine the number of points to select: $N = \min(M, \text{n\_points})$ [S1, unofficial].
2. Generate a random permutation of indices $P$ of length $M$, containing each integer from $0$ to $M-1$ exactly once [S1, unofficial].
3. Select the first $N$ indices from $P$: $I = P[0:N]$ [S1, unofficial].
4. Index the `points` tensor with $I$ to retrieve the shuffled points [S1, unofficial].
5. Reshape the result to ensure a 2D layout of shape $(-1, 3)$ [S1, unofficial].
6. Return the resulting tensor [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def random_point_subsampling(points: torch.Tensor, n_points: int) -> torch.Tensor:
    """
    Selects a random subset of points without replacement.
    
    Args:
        points: Tensor of shape (M, 3).
        n_points: Number of points to select.
        
    Returns:
        Tensor of shape (N, 3) where N <= n_points.
    """
    M = points.shape[0]
    # Clamp n_points to available points to avoid index errors
    N = min(M, n_points)
    
    # Generate random permutation of indices [0, M-1]
    # [G] torch.randperm ensures no replacement and uniform distribution
    perm = torch.randperm(M, device=points.device)
    
    # Select first N indices
    selected_indices = perm[:N]
    
    # Gather points
    sampled_points = points[selected_indices]
    
    # Ensure 2D shape (-1, 3)
    sampled_points = sampled_points.view(-1, 3)
    
    return sampled_points
```

**Parameters:**
- `n_points`: The target number of points. Typical values depend on the downstream voxelization resolution, often matching the target voxel count [S1, unofficial].

**Edge Cases:**
- If `n_points` > $M$, the method returns all $M$ points [S1, unofficial].
- If `points` is empty, the result is an empty tensor of shape $(0, 3)$ [S1, unofficial].

## Subtopics

### Near Sampling
A site placement strategy where a proportion of sites are placed close to uniformly subsampled input point cloud data [S2]. It is used in methods like DCCVT to initialize site distributions before optimization [S2]. It is chosen when a hybrid approach of uniform grid initialization and point-cloud proximity is desired [S2]. It differs from pure random subsampling by biasing selection toward existing point locations rather than uniform space [S2].

## Operating characteristics
**Time Complexity:** $O(M)$ to generate the permutation and $O(N)$ to gather indices, where $M$ is the input size and $N$ is the output size [S1, unofficial].
**Memory Complexity:** $O(M)$ to store the permutation array [S1, unofficial].
**Strengths:** Simple to implement; unbiased selection; fast execution [S1, unofficial].
**Limitations:** Ignores geometric structure; may result in clusters or gaps in the output distribution; not suitable for preserving surface detail [S1, unofficial].
**When to use:** As a baseline for resampling; when computational speed is critical and geometric uniformity is not required [S1, unofficial].
**When not to use:** When preserving local density or surface features is important [S1, unofficial].

## Background
The method is implemented in the Neural Surfaces repository by Omkar Chittar [S1, unofficial]. It is used in training pipelines for neural implicit surface reconstruction, specifically within the optimization loop for models like VolSDF [S1, unofficial]. The technique is a standard statistical sampling method applied to point cloud data in computer graphics [S1, unofficial].

## Sources
[S1] omkarchittar/Neural_Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
[S2] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
