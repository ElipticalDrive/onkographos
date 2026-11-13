# Farthest Point Sampling (FPS)

Also called: Furthest Point Sampling (FPS), KPConv, Point Cloud Downsampling via Farthest Point Sampling

## Overview
Farthest Point Sampling (FPS) is a deterministic (after initial random seed) algorithm for selecting a subset of points from a point cloud such that the selected points are maximally spread out in Euclidean space, ensuring maximum spatial coverage [S1][S2, unofficial]. It solves the problem of uniform downsampling where simple random sampling may leave gaps or cluster points, and grid-based methods may lose fine geometric details. It sets itself apart from sibling methods like [Adaptive Point Cloud Downsampling based on Geometry and Texture Complexity](10729-adaptive-point-cloud-downsampling-based-on-geometry-and-text.md) by relying purely on geometric distance metrics rather than local feature complexity or texture density.

## How it works
**Inputs:**
- `points`: A numpy array of shape $(N, 3)$ containing the 3D coordinates of $N$ points in the cloud.
- `sample_count`: An integer specifying the target number of points to select.

**Outputs:**
- `indices`: A numpy array of shape $(sample\_count,)$ containing the indices of the selected points from the original `points` array.

**Data Structures:**
- `points_index`: Array of integers $[0, 1, \dots, N-1]$.
- `A`: Array of selected indices.
- `B`: Array of candidate indices (initially all indices not in `A`).
- `min_dis_B2A`: Array of floats, same length as `B`, storing the squared Euclidean distance from each point in `B` to its nearest neighbor in `A`.

**Algorithm Steps:**
1. Initialize `points_index` as integers from $0$ to $N-1$.
2. Select one random index from `points_index` as the first element of set `A`.
3. Set `B` to contain all indices in `points_index` except those in `A`.
4. Compute the squared Euclidean distance from each point in `B` to the single point in `A` (index `A[0]`). Store these distances in `min_dis_B2A`.
   - Note: The distance function must correctly compute differences in all three dimensions ($x, y, z$). A common implementation error is ignoring the $z$-axis difference [S2, unofficial].
5. While the size of `A` is less than `sample_count`:
   a. Find the index in `B` with the maximum value in `min_dis_B2A`. This is the point in `B` furthest from the current set `A`.
   b. Add this point's index to `A` and remove it from `B` and `min_dis_B2A`.
   c. Update `min_dis_B2A`: For each remaining point in `B`, compute its squared Euclidean distance to the newly added point in `A` (the last element of `A`). Update the stored distance to be the minimum of the old distance and the new distance.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import math

def distance_point3d(p0, p1):
    """
    Computes the squared Euclidean distance between two 3D points.
    p0, p1: arrays of shape (3,)
    """
    # Correct implementation: (x0-x1)^2 + (y0-y1)^2 + (z0-z1)^2
    # Pitfall: Do not use (p0[2] - p0[2])**2 which ignores Z-axis [S2, unofficial]
    dx = p0[0] - p1[0]
    dy = p0[1] - p1[1]
    dz = p0[2] - p1[2]
    return dx*dx + dy*dy + dz*dz

def furthest_point_sample(points, sample_count):
    """
    Selects sample_count points from points using Farthest Point Sampling.
    
    Args:
        points: numpy array of shape (N, 3)
        sample_count: int, number of points to sample
        
    Returns:
        indices: numpy array of shape (sample_count,)
    """
    N = points.shape[0]
    if sample_count > N:
        sample_count = N
        
    # Initialize index array
    points_index = np.arange(N, dtype=np.int64)
    
    # Select one random index as the first element of A
    A = np.array([np.random.choice(points_index)])
    
    # B contains all indices except those in A
    B = np.setdiff1d(points_index, A)
    
    # Compute initial distances from B to A[0]
    # min_dis_B2A[i] stores the distance from B[i] to the nearest point in A
    min_dis_B2A = np.zeros(len(B))
    for i in range(len(B)):
        Pa = points[A[0]]
        Pb = points[B[i]]
        min_dis_B2A[i] = distance_point3d(Pa, Pb)
    
    # Iteratively select the farthest point
    while len(A) < sample_count:
        # Find the index in B with the maximum distance to A
        longest_points_in_B_index = np.argmax(min_dis_B2A)
        longest_points_index = B[longest_points_in_B_index]
        
        # Update A and B
        A = np.append(A, longest_points_index)
        B = np.delete(B, longest_points_in_B_index)
        min_dis_B2A = np.delete(min_dis_B2A, longest_points_in_B_index)
        
        # Update min_dis_B2A with the new point in A
        new_point_in_A = points[A[-1]]
        for i in range(len(B)):
            Pb = points[B[i]]
            dist = distance_point3d(new_point_in_A, Pb)
            # Keep the minimum distance to any point in A
            min_dis_B2A[i] = min(dist, min_dis_B2A[i])
            
    return A
```

**Parameters:**
- `sample_count`: The target number of points to select. Typical values depend on the application, e.g., 4096 points for downsampling [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Distance Calculation Bug:** A common implementation error in the distance function is calculating `(p0[2] - p0[2])**2` instead of `(p0[2] - p1[2])**2`, effectively ignoring the Z-axis difference in the squared distance calculation [S2, unofficial].
- **Precision:** Using squared distances avoids square roots during comparisons, which is efficient and numerically stable for finding the maximum. However, final distances should be computed if needed.
- **Boundary Conditions:** If `sample_count` is greater than the number of points, the algorithm should handle this gracefully (e.g., return all points).

## Subtopics

### [Farthest Point Sampling Initialization](13863-farthest-point-sampling-initialization.md)
Farthest Point Sampling Initialization is a method for selecting a subset of points from a point cloud such that they are maximally spread out in Euclidean space, ensuring maximum spatial coverage. It serves as the primary initialization step for primitive positions in neural rendering representations, replacing simpler nearest-neighbor heuristics to prevent the formation of overly small primitives. It builds on [Farthest Point Sampling (FPS)](7182-farthest-point-sampling-fps.md) by providing the specific iterative selection logic and CUDA-optimized implementation details required to generate the initial set of indices.

## Operating characteristics
- **Time Complexity:** $O(N \cdot K)$, where $N$ is the number of points and $K$ is the number of sampled points. In each iteration, we compute distances for all remaining points in $B$ to the newly added point in $A$.
- **Memory Complexity:** $O(N)$ to store the indices and distances.
- **Measured Performance:** Used to downsample point clouds to 4096 points, splitting them into 90% center points and 10% edge points [S2, unofficial].
- **Strengths:** Ensures uniform spatial coverage; simple to implement; deterministic after initial random seed.
- **Limitations:** Computationally expensive for large $N$ and $K$; sensitive to the initial random point; does not consider local geometric features or density variations.
- **When to Use:** When uniform spatial distribution is required, such as initializing neural rendering primitives or downsampling point clouds for further processing.
- **When Not to Use:** When local geometric features or density variations are important, or when computational efficiency is critical for very large point clouds.

## Background
Farthest Point Sampling is a fundamental technique in point cloud processing and 3D representation learning. It is used in various contexts, including mesh generation [S1] and point-based neural rendering [S1]. The method is often employed in conjunction with other downsampling techniques like grid subsampling or partition-based grid pooling to ensure robustness to variation in point density [S1]. It is also used in the context of KPConv, a convolutional network for point clouds [S1].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S2] jsLi1/Voxelization, https://github.com/jsLi1/Voxelization, fetched 2026-10-27 (unofficial)
