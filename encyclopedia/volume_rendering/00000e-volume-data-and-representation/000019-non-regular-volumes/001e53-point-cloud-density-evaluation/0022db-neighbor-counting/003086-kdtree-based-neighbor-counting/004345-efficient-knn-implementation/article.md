# Efficient kNN Implementation

Also called: Customized CUDA kNN, Local kNN with Camera Parameters

## Overview
Efficient kNN Implementation is a method for accelerating k-nearest neighbor searches in point cloud and Gaussian splatting rendering pipelines [S1]. It solves the $O(N^2)$ computational bottleneck of global brute-force kNN queries by providing two distinct strategies: a high-performance global search for moderate datasets and a scalable local search for large-scale scenes [S1]. It sets itself apart from its parent [KDTree-based Neighbor Counting](6285-kdtree-based-neighbor-counting.md) by leveraging camera parameters to restrict search spaces [S1].

## How it works
The method provides two implementations based on dataset size [S1].

**Inputs:**
- Point cloud $P$ with $N$ points [S1].
- Integer $k$ for the number of neighbors [S1].
- Camera parameters (intrinsics and extrinsics) for the scene views [S1].

**Outputs:**
- Indices of the $k$ nearest neighbors for each point in $P$ [S1].
- Squared distances to these neighbors [S1].

**Data Structures:**
- Global kNN: Uses a customized CUDA kernel structure for batched processing [S1].
- Local kNN: Uses a candidate set $C$ of size $C_{size}$ [S1].

**Steps for Global kNN (Small Scale):**
1. For point counts $N < 300,000$, invoke a customized CUDA implementation of global kNN [S1].
2. Compute exact 3D Euclidean distances between all pairs or use optimized GPU parallel reduction to find the top-$k$ points [S1].
3. Return indices and squared distances [S1].

**Steps for Local kNN (Large Scale):**
1. If $N \ge 300,000$ or view counts are high, switch to the efficient local kNN alternative [S1].
2. Construct a candidate set $C$ for each query point using known camera parameters [S1]:
   - Aggregate spatial neighbors from the same view [S1].
   - Project points from nearby cameras into the current view to identify cross-view neighbors [S1].
3. Compute exact 3D distances strictly within this constant-sized candidate set $C$ [S1].
4. Select the top-$k$ points from $C$ based on these distances [S1].
5. Return indices and squared distances [S1].

**Parameters:**
- Threshold for switching strategies: $N = 300,000$ points [S1].
- Candidate set size: $C$ (constant, value not given in sources) [S1].

**Edge Cases and Pitfalls:**
- Precision: Exact 3D distances are computed within the candidate set to guarantee correctness [S1].
- Boundaries: The local method relies on the accuracy of camera parameters; incorrect projections may exclude true neighbors [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def efficient_knn(points, k, camera_params, threshold=300000):
    N = len(points)
    if N < threshold:
        # Global kNN for small datasets [S1]
        # Uses customized CUDA implementation [S1]
        # Standard GPU-accelerated search [G]
        indices, distances = _global_knn_impl(points, k)
        return indices, distances
    else:
        all_indices = []
        all_distances = []
        for i in range(N):
            query = points[i]
            candidates = set()
            # 1. Aggregate spatial neighbors from the same view [S1]
            # Not given in the sources. [G]
            # candidates.update(_same_view_logic(points, i, camera_params))
            # 2. Aggregate cross-view neighbors projected from nearby cameras [S1]
            # Not given in the sources. [G]
            # candidates.update(_cross_view_logic(points, i, camera_params))
            candidates.discard(i)
            dists = []
            for cand_idx in candidates:
                cand = points[cand_idx]
                dist_sq = sum((query[j] - cand[j])**2 for j in range(3))
                dists.append((cand_idx, dist_sq))
            dists.sort(key=lambda x: x[1])
            top_k = dists[:k]
            all_indices.append([idx for idx, _ in top_k])
            all_distances.append([d for _, d in top_k])
        return all_indices, all_distances

def _global_knn_impl(points, k):
    # Not given in the sources. [G]
    return [], []
```

## Subtopics

### [Efficient Local kNN](10975-efficient-local-knn.md)
Efficient Local kNN is a drop-in replacement for global brute-force k-nearest neighbors (kNN) search in point cloud or Gaussian splatting rendering pipelines. It solves the $O(N^2)$ complexity bottleneck of global search by exploiting the multi-view grid structure of the scene, reducing complexity to $O(N)$. It sets itself apart from global kNN by constraining the search space to spatially adjacent pixels in the source view and corresponding projected pixels in other views, achieving significant speedups with negligible quality loss. See [Efficient Local kNN](10975-efficient-local-knn.md).

### [PointOps2 CUDA Kernels](13912-pointops2-cuda-kernels.md)
PointOps2 CUDA Kernels are a collection of GPU-accelerated operations for point cloud processing, exposed via a PyTorch C++ extension. This specific article covers the **Point Subtraction** operation, which computes the difference between feature vectors of neighboring points and a reference point. It solves the problem of efficiently calculating relative positional or feature offsets in non-regular volumes, a prerequisite for subsequent aggregation or interpolation steps. It differs from sibling methods like [Efficient Local kNN](10975-efficient-local-knn.md) by focusing on the arithmetic difference of features rather than just the spatial search, and from [PointOps2 Aggregation Kernels](13922-pointops2-aggregation-kernels.md) by performing element-wise subtraction rather than weighted summation. See [PointOps2 CUDA Kernels](13912-pointops2-cuda-kernels.md).

### [K-Nearest Neighbors Query](13918-k-nearest-neighbors-query.md)
A method to find the $k$ nearest neighbors for each point in a target set within a source set, returning indices and squared distances. It performs K-Nearest Neighbors search on 3D points using a CUDA kernel, handling batched inputs with offsets. It differs from [Efficient Local kNN](10975-efficient-local-knn.md) by focusing on global batched queries with explicit offset management, and from [PointOps2 CUDA Kernels](13912-pointops2-cuda-kernels.md) by being a specific high-level API wrapper rather than a low-level kernel library. See [K-Nearest Neighbors Query](13918-k-nearest-neighbors-query.md).

### Reciprocal Nearest Neighbor Matching
Finds mutual nearest neighbors between two point sets $P1$ and $P2$ using KDTree to identify reciprocal matches [S2, unofficial]. Build KDTree for $P1$ and $P2$ [S2, unofficial]. Query tree2 for $P1$ to get $nn1\_in\_P2$ (indices of closest points in $P2$ for each point in $P1$) [S2, unofficial]. Query tree1 for $P2$ to get $nn2\_in\_P1$ (indices of closest points in $P1$ for each point in $P2$) [S2, unofficial]. Use workers=8 [S2, unofficial]. Compute $reciprocal\_in\_P1 = (nn2\_in\_P1[nn1\_in\_P2] == np.arange(len(nn1\_in\_P2)))$ [S2, unofficial]. Compute $reciprocal\_in\_P2 = (nn1\_in\_P2[nn2\_in\_P1] == np.arange(len(nn2\_in\_P1)))$ [S2, unofficial]. Assert sums are equal [S2, unofficial]. Return $reciprocal\_in\_P2$, $nn2\_in\_P1$, sum [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Global kNN has $O(N^2)$ complexity [S1]. Efficient local kNN reduces complexity to $O(N \cdot C)$ where $C$ is the candidate set size [S1].
- **Memory Complexity:** Not given in the sources [S1].
- **Measured Performance:** For point counts < 300K, the customized CUDA implementation is used [S1]. For higher resolutions or view counts, the local kNN alternative is required to avoid bottlenecks [S1].
- **Scaling:** The local kNN method scales efficiently to high-resolution images and large point clouds by restricting the search space [S1].
- **Strengths:** High performance for small datasets via CUDA; scalability for large datasets via camera-parameter-guided local search [S1].
- **Limitations:** Global kNN becomes a bottleneck at high resolutions/view counts [S1]. Local kNN depends on accurate camera parameters [S1].
- **When to use:** Use global kNN for $N < 300,000$ points [S1]. Use local kNN for larger datasets or multi-view scenes [S1].

## Background
The method originates from the ReSplat project, which focuses on learning recurrent Gaussian splatting [S1]. It was developed to address the computational limitations of global kNN searches in large-scale 3D Gaussian splatting pipelines [S1]. The reciprocal nearest neighbor matching variant is associated with the InstantSplat project [S2, unofficial].

## Sources
[S1] ReSplat: Learning Recurrent Gaussian Splatting, https://arxiv.org/abs/2510.08575v4, fetched 2026-10-26
[S2] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
