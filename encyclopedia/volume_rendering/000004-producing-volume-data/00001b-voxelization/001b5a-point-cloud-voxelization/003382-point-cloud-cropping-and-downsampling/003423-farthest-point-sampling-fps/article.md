# Farthest Point Sampling (FPS)

Also called: Furthest Point Sampling (FPS), Point Cloud Downsampling via Farthest Point Sampling

## Overview
Farthest Point Sampling (FPS) is a stochastic algorithm for selecting a subset of points from a point cloud such that the selected points are maximally spread out in Euclidean space, ensuring maximum spatial coverage [S2, unofficial]. It solves the problem of uniform downsampling by iteratively picking the point furthest from the currently selected set [S2, unofficial].

## How it works
**Inputs:**
- `points`: An array of shape $(N, 3)$ containing the 3D coordinates of $N$ points [S2, unofficial].
- `sample_count`: An integer specifying the target number of points to select [S2, unofficial].

**Outputs:**
- `indices`: An array of shape $(sample\_count,)$ containing the indices of the selected points from the original `points` array [S2, unofficial].

**Data Structures:**
- `points_index`: An array of integers $[0, 1, \dots, N-1]$ [S2, unofficial].
- `A`: An array of selected indices [S2, unofficial].
- `B`: An array of candidate indices, initially containing all indices not in `A` [S2, unofficial].
- `min_dis_B2A`: An array of floats, same length as `B`, storing the squared Euclidean distance from each point in `B` to its nearest neighbor in `A` [S2, unofficial].

**Algorithm Steps:**
1. Initialize `points_index` as integers from $0$ to $N-1$ [S2, unofficial].
2. Select one random index from `points_index` as the first element of set `A` [S2, unofficial].
3. Set `B` to contain all indices in `points_index` except those in `A` [S2, unofficial].
4. Compute the squared Euclidean distance from each point in `B` to the single point in `A` (index `A[0]`) and store these distances in `min_dis_B2A` [S2, unofficial].
5. While the size of `A` is less than `sample_count`:
   a. Find the index in `B` with the maximum value in `min_dis_B2A`, which corresponds to the point in `B` furthest from the current set `A` [S2, unofficial].
   b. Add this point's index to `A` and remove it from `B` and `min_dis_B2A` [S2, unofficial].
   c. Update `min_dis_B2A`: For each remaining point in `B`, compute its squared Euclidean distance to the newly added point in `A` (the last element of `A`), and update the stored distance to be the minimum of the old distance and the new distance [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import math

def distance_point3d_sq(p0, p1):
    dx = p0[0] - p1[0]
    dy = p0[1] - p1[1]
    dz = p0[2] - p1[2]
    return dx * dx + dy * dy + dz * dz

def furthest_point_sample(points, sample_count):
    N = points.shape[0]
    if sample_count > N:
        sample_count = N
    points_index = np.arange(N, dtype=np.int64)
    A = np.array([np.random.choice(points_index)])
    B = np.setdiff1d(points_index, A)
    min_dis_B2A = np.zeros(len(B))
    for i in range(len(B)):
        min_dis_B2A[i] = distance_point3d_sq(points[A[0]], points[B[i]])
    while len(A) < sample_count:
        longest_points_in_B_index = np.argmax(min_dis_B2A)
        longest_points_index = B[longest_points_in_B_index]
        A = np.append(A, longest_points_index)
        B = np.delete(B, longest_points_in_B_index)
        min_dis_B2A = np.delete(min_dis_B2A, longest_points_in_B_index)
        new_point_in_A = points[A[-1]]
        for i in range(len(B)):
            dist = distance_point3d_sq(new_point_in_A, points[B[i]])
            min_dis_B2A[i] = min(dist, min_dis_B2A[i])
    return A
```

**Parameters:**
- `sample_count`: The target number of points to select [S2, unofficial].

**Edge Cases and Pitfalls:**
- Distance Calculation Bug: A common implementation error calculates `(p0[2] - p0[2])**2` instead of `(p0[2] - p1[2])**2`, effectively ignoring the Z-axis difference in the squared distance calculation [S2, unofficial].
- Precision: Using squared distances avoids square roots during comparisons, which is efficient and numerically stable for finding the maximum [G].
- Boundary Conditions: If `sample_count` exceeds $N$, the algorithm should clamp it to $N$ to avoid index errors [G].

## Subtopics

### [Farthest Point Sampling Initialization](13863-farthest-point-sampling-initialization.md)
Farthest Point Sampling Initialization is a method for selecting a subset of points from a point cloud such that they are maximally spread out in Euclidean space, ensuring maximum spatial coverage. It serves as the primary initialization step for primitive positions in neural rendering representations, replacing simpler nearest-neighbor heuristics to prevent the formation of overly small primitives. It builds on [Farthest Point Sampling (FPS)](7182-farthest-point-sampling-fps.md) by providing the specific iterative selection logic and CUDA-optimized implementation details required to generate the initial set of indices.

## Operating characteristics
- Time Complexity: $O(N \cdot K)$, where $N$ is the number of points and $K$ is the number of sampled points [G].
- Memory Complexity: $O(N)$ to store the indices and distances [G].
- Strengths: Ensures uniform spatial coverage and is simple to implement [S2, unofficial].
- Limitations: Computationally expensive for large $N$ and $K$, and sensitive to the initial random point [G].
- When to Use: When uniform spatial distribution is required for downsampling [S2, unofficial].
- When Not to Use: When local geometric features or density variations are important [G].

## Background
Not given in the sources.

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S2] jsLi1/Voxelization, https://github.com/jsLi1/Voxelization, fetched 2026-10-27 (unofficial)
