# Efficient kNN Implementation

Also called: Customized CUDA kNN, Local kNN with Camera Parameters

## Overview
Efficient kNN Implementation is a method for accelerating k-nearest neighbor searches in point cloud and Gaussian splatting rendering pipelines [S1]. It solves the $O(N^2)$ computational bottleneck of global brute-force kNN queries by providing two distinct strategies: a high-performance global search for moderate datasets and a scalable local search for large-scale scenes [S1]. It sets itself apart from sibling methods like [Kernel Density Estimation (KDE)](13917-kernel-density-estimation-kde.md) by focusing on exact neighbor indexing rather than density smoothing, and from its parent [KDTree-based Neighbor Counting](6285-kdtree-based-neighbor-counting.md) by leveraging camera parameters to restrict search spaces [S1].

## How it works
The method provides two implementations based on dataset size.

**Inputs:**
- Point cloud $P$ with $N$ points.
- Integer $k$ for the number of neighbors.
- Camera parameters (intrinsics and extrinsics) for the scene views [S1].

**Outputs:**
- Indices of the $k$ nearest neighbors for each point in $P$.
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
    """
    points: List of 3D coordinates, shape (N, 3)
    k: int, number of neighbors
    camera_params: Dict or object containing intrinsics and extrinsics for views
    threshold: int, point count threshold for switching strategies
    """
    N = len(points)
    
    if N < threshold:
        # Global kNN for small datasets
        # Uses customized CUDA implementation [S1]
        # Implementation details of the CUDA kernel are not provided in sources [G]
        # Assuming a standard GPU-accelerated brute-force or optimized search
        indices, distances = global_knn_cuda(points, k)
        return indices, distances
    else:
        # Local kNN for large datasets [S1]
        indices = []
        distances = []
        
        for i in range(N):
            query_point = points[i]
            candidate_indices = []
            candidate_distances = []
            
            # Construct candidate set C using camera parameters [S1]
            # 1. Aggregate spatial neighbors from the same view
            same_view_neighbors = get_spatial_neighbors_same_view(points, i, camera_params)
            candidate_indices.extend(same_view_neighbors)
            
            # 2. Aggregate cross-view neighbors projected from nearby cameras
            cross_view_neighbors = get_cross_view_neighbors(points, i, camera_params)
            candidate_indices.extend(cross_view_neighbors)
            
            # Remove duplicates if any
            candidate_indices = list(set(candidate_indices))
            
            # Compute exact 3D distances within this set [S1]
            for cand_idx in candidate_indices:
                cand_point = points[cand_idx]
                dist_sq = sum((query_point[j] - cand_point[j])**2 for j in range(3))
                candidate_distances.append((dist_idx, dist_sq))
            
            # Sort by distance and select top-k [S1]
            candidate_distances.sort(key=lambda x: x[1])
            top_k = candidate_distances[:k]
            
            indices.append([idx for idx, _ in top_k])
            distances.append([dist for _, dist in top_k])
            
        return indices, distances

def global_knn_cuda(points, k):
    """
    Placeholder for customized CUDA implementation [S1].
    Returns indices and squared distances.
    """
    # Not given in the sources. [G]
    pass

def get_spatial_neighbors_same_view(points, query_idx, camera_params):
    """
    Aggregates spatial neighbors from the same view [S1].
    Not given in the sources. [G]
    """
    pass

def get_cross_view_neighbors(points, query_idx, camera_params):
    """
    Projects points from nearby cameras to find cross-view neighbors [S1].
    Not given in the sources. [G]
    """
    pass
```

## Subtopics

### [Efficient Local kNN](10975-efficient-local-knn.md)
Efficient Local kNN is a drop-in replacement for global brute-force k-nearest neighbors (kNN) search in point cloud or Gaussian splatting rendering pipelines. It solves the $O(N^2)$ complexity bottleneck of global search by exploiting the multi-view grid structure of the scene, reducing complexity to $O(N)$. It sets itself apart from global kNN by constraining the search space to spatially adjacent pixels in the source view and corresponding projected pixels in other views, achieving significant speedups with negligible quality loss. See [Efficient Local kNN](10975-efficient-local-knn.md).

### [PointOps2 CUDA Kernels](13912-pointops2-cuda-kernels.md)
PointOps2 CUDA Kernels are a collection of GPU-accelerated operations for point cloud processing, exposed via a PyTorch C++ extension. This specific article covers the **Point Subtraction** operation, which computes the difference between feature vectors of neighboring points and a reference point. It solves the problem of efficiently calculating relative positional or feature offsets in non-regular volumes, a prerequisite for subsequent aggregation or interpolation steps. It differs from sibling methods like [Efficient Local kNN](10975-efficient-local-knn.md) by focusing on the arithmetic difference of features rather than just the spatial search, and from [PointOps2 Aggregation Kernels](13922-pointops2-aggregation-kernels.md) by performing element-wise subtraction rather than weighted summation. See [PointOps2 CUDA Kernels](13912-pointops2-cuda-kernels.md).

### [K-Nearest Neighbors Query](13918-k-nearest-neighbors-query.md)
A method to find the $k$ nearest neighbors for each point in a target set within a source set, returning indices and squared distances. It performs K-Nearest Neighbors search on 3D points using a CUDA kernel, handling batched inputs with offsets. It differs from [Efficient Local kNN](10975-efficient-local-knn.md) by focusing on global batched queries with explicit offset management, and from [PointOps2 CUDA Kernels](13912-pointops2-cuda-kernels.md) by being a specific high-level API wrapper rather than a low-level kernel library. See [K-Nearest Neighbors Query](13918-k-nearest-neighbors-query.md).

### Reciprocal Nearest Neighbor Matching
Finds mutual nearest neighbors between two point sets $P1$ and $P2$ using KDTree to identify reciprocal matches [S2, unofficial]. Build KDTree for $P1$ and $P2$. Query tree2 for $P1$ to get $nn1\_in\_P2$ (indices of closest points in $P2$ for each point in $P1$). Query tree1 for $P2$ to get $nn2\_in\_P1$ (indices of closest points in $P1$ for each point in $P2$). Use workers=8. Compute $reciprocal\_in\_P1 = (nn2\_in\_P1[nn1\_in\_P2] == np.arange(len(nn1\_in\_P2)))$. Compute $reciprocal\_in\_P2 = (nn1\_in\_P2[nn2\_in\_P1] == np.arange(len(nn2\_in\_P1)))$. Assert sums are equal. Return $reciprocal\_in\_P2$, $nn2\_in\_P1$, sum [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Global kNN has $O(N^2)$ complexity [S1]. Efficient local kNN reduces complexity to $O(N \cdot C)$ where $C$ is the candidate set size [S1].
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** For point counts < 300K, the customized CUDA implementation is used [S1]. For higher resolutions or view counts, the local kNN alternative is required to avoid bottlenecks [S1].
- **Scaling:** The local kNN method scales efficiently to high-resolution images and large point clouds by restricting the search space [S1].
- **Strengths:** High performance for small datasets via CUDA; scalability for large datasets via camera-parameter-guided local search [S1].
- **Limitations:** Global kNN becomes a bottleneck at high resolutions/view counts [S1]. Local kNN depends on accurate camera parameters [S1].
- **When to use:** Use global kNN for $N < 300,000$ points. Use local kNN for larger datasets or multi-view scenes [S1].

## Background
The method originates from the ReSplat project, which focuses on learning recurrent Gaussian splatting [S1]. It was developed to address the computational limitations of global kNN searches in large-scale 3D Gaussian splatting pipelines [S1]. The reciprocal nearest neighbor matching variant is associated with the InstantSplat project [S2, unofficial].

## Sources
[S1] ReSplat: Learning Recurrent Gaussian Splatting, https://arxiv.org/abs/2510.08575v4, fetched 2026-10-26
[S2] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
