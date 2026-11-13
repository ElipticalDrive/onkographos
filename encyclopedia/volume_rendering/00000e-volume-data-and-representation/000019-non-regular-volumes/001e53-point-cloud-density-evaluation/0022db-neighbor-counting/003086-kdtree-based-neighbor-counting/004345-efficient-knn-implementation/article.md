# Efficient kNN Implementation

Also called: Customized CUDA kNN, Local kNN with Camera Parameters

## Overview
Efficient kNN Implementation provides scalable nearest-neighbor search for point cloud density evaluation in volume rendering, specifically addressing the computational bottleneck of global $O(N^2)$ searches in large-scale scenes [S1]. It distinguishes itself from standard global kNN by dynamically selecting between a brute-force CUDA implementation for small datasets and a camera-parameter-constrained local search for large datasets, thereby reducing complexity to $O(N \cdot C)$ where $C$ is a small candidate set size [S1]. This method serves as the core neighbor-counting engine for [KDTree-based Neighbor Counting](6285-kdtree-based-neighbor-counting.md), enabling efficient density estimation for mesh vertices or Gaussian splats [S1].

## How it works
The method implements a hybrid strategy that selects the search algorithm based on the total point count $N$. It takes a set of 3D points and camera parameters as input and returns the indices and distances of the $k$ nearest neighbors for each point.

### Data Structures
*   **Point Cloud**: A list or tensor of $N$ points, where each point is a 3D vector $(x, y, z)$.
*   **Camera Parameters**: For each view, the extrinsic matrix (rotation $R$ and translation $t$) and intrinsic parameters (focal length, principal point) are required to project points to 2D screen space [S1].
*   **Candidate Set**: A subset of points selected for detailed distance calculation, limited to size $C$ per query point [S1].
*   **Output**: For each query point, a list of $k$ indices corresponding to the nearest neighbors and their squared Euclidean distances.

### Steps
1.  **Check Scale**: Determine if the total number of points $N$ is less than 300,000 [S1].
2.  **Global Search (Small Scale)**: If $N < 300,000$, use a customized CUDA implementation of global kNN [S1].
    *   This approach computes distances to all other points, resulting in $O(N^2)$ complexity [S1].
    *   It is suitable for smaller scenes where the overhead of candidate selection outweighs the benefit [S1].
3.  **Local Search (Large Scale)**: If $N \ge 300,000$, use the efficient local kNN alternative [S1].
    *   **Construct Candidate Set**: For each query point, identify a candidate set of size $C$ [S1].
        *   **Same-View Neighbors**: Identify points that are spatially adjacent in the current source view [S1].
        *   **Cross-View Neighbors**: Project points from nearby cameras into the current view's coordinate system and select those that are spatially close to the query point's projection [S1].
    *   **Distance Calculation**: Compute exact 3D Euclidean distances between the query point and all points in the candidate set [S1].
    *   **Top-k Selection**: Sort the candidate set by distance and select the top $k$ neighbors [S1].
4.  **Return Results**: Output the indices and distances for the $k$ nearest neighbors.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def efficient_knn(points, camera_params, k, threshold=300000):
    """
    Efficient kNN implementation for point cloud density evaluation.
    
    Args:
        points: List of 3D points, shape (N, 3).
        camera_params: List of camera parameters for each view.
        k: Number of nearest neighbors to find.
        threshold: Point count threshold for switching strategies.
        
    Returns:
        indices: Indices of k nearest neighbors for each point, shape (N, k).
        distances: Squared distances to k nearest neighbors, shape (N, k).
    """
    N = len(points)
    
    if N < threshold:
        # Global kNN for small point clouds
        # Uses customized CUDA implementation [S1]
        # Complexity: O(N^2)
        indices, distances = global_knn_cuda(points, k)
        return indices, distances
    else:
        # Local kNN for large point clouds
        # Complexity: O(N * C) where C is candidate set size [S1]
        indices = []
        distances = []
        
        for i, point in enumerate(points):
            # Construct candidate set using camera parameters [S1]
            candidates = []
            
            # 1. Aggregate spatial neighbors from the same view [S1]
            same_view_neighbors = get_spatial_neighbors_same_view(point, camera_params, i)
            candidates.extend(same_view_neighbors)
            
            # 2. Aggregate cross-view neighbors projected from nearby cameras [S1]
            cross_view_neighbors = get_cross_view_neighbors(point, camera_params, i)
            candidates.extend(cross_view_neighbors)
            
            # Remove duplicates and limit to constant size C [S1]
            unique_candidates = remove_duplicates(candidates)
            candidate_set = unique_candidates[:C]
            
            # Compute exact 3D distances within the candidate set [S1]
            dists = []
            for cand in candidate_set:
                dist = euclidean_distance_squared(point, cand)
                dists.append((dist, cand.index))
            
            # Sort by distance and select top-k [S1]
            dists.sort(key=lambda x: x[0])
            top_k = dists[:k]
            
            indices.append([idx for _, idx in top_k])
            distances.append([dist for dist, _ in top_k])
            
        return indices, distances

def global_knn_cuda(points, k):
    """
    Customized CUDA implementation of global kNN [S1].
    """
    # Call CUDA kernel [S1]
    # Returns indices and squared distances
    return cuda_knn_kernel(points, k)

def get_spatial_neighbors_same_view(point, camera_params, point_idx):
    """
    Get spatially adjacent points in the same view [S1].
    """
    # Project point to 2D screen space using camera parameters
    # Find neighbors in 2D grid or spatial hash
    pass

def get_cross_view_neighbors(point, camera_params, point_idx):
    """
    Get neighbors projected from nearby cameras [S1].
    """
    # For each nearby camera, project point to its 2D space
    # Find corresponding points in other views
    pass

def euclidean_distance_squared(p1, p2):
    """
    Compute squared Euclidean distance between two 3D points.
    """
    return sum((a - b) ** 2 for a, b in zip(p1, p2))

def remove_duplicates(candidates):
    """
    Remove duplicate points from the candidate list.
    """
    seen = set()
    unique = []
    for cand in candidates:
        if cand.index not in seen:
            seen.add(cand.index)
            unique.append(cand)
    return unique
```

### Parameters
*   **$N$**: Total number of points. Typical threshold for switching strategies is 300,000 [S1].
*   **$k$**: Number of nearest neighbors. Not given in the sources.
*   **$C$**: Size of the candidate set. Not given in the sources, but described as "small" and "constant-sized" [S1].
*   **Camera Parameters**: Extrinsic and intrinsic matrices for each view. Required for local kNN [S1].

### Edge Cases and Pitfalls
*   **Precision**: Use squared distances to avoid square root computations during sorting, then take the square root only if needed [S3, unofficial].
*   **Boundaries**: Ensure that candidate set construction handles edge cases where points are near the boundary of the view frustum [S1].
*   **Degenerate Input**: If no neighbors are found in the candidate set, return an empty list or handle gracefully [G].

## Subtopics

### [Efficient Local kNN](10975-efficient-local-knn.md)
Efficient Local kNN is a drop-in replacement for global brute-force k-nearest neighbors (kNN) search in point cloud or Gaussian splatting rendering pipelines. It solves the $O(N^2)$ complexity bottleneck of global search by exploiting the multi-view grid structure of the scene, reducing complexity to $O(N)$. It sets itself apart from global kNN by constraining the search space to spatially adjacent pixels in the source view and corresponding projected pixels in other views, achieving significant speedups with negligible quality loss. See [Efficient Local kNN](10975-efficient-local-knn.md).

### [PointOps2 CUDA Kernels](13912-pointops2-cuda-kernels.md)
PointOps2 CUDA Kernels are a collection of GPU-accelerated operations for point cloud processing, exposed via a PyTorch C++ extension. This specific article covers the **Point Subtraction** operation, which computes the difference between feature vectors of neighboring points and a reference point. It solves the problem of efficiently calculating relative positional or feature offsets in non-regular volumes, a prerequisite for subsequent aggregation or interpolation steps. It differs from sibling methods like [Efficient Local kNN](10975-efficient-local-knn.md) by focusing on the arithmetic difference of features rather than just the spatial search, and from [PointOps2 Aggregation Kernels](13922-pointops2-aggregation-kernels.md) by performing element-wise subtraction rather than weighted summation. See [PointOps2 CUDA Kernels](13912-pointops2-cuda-kernels.md).

### Reciprocal Nearest Neighbor Matching
Reciprocal Nearest Neighbor Matching finds mutual nearest neighbors between two point sets $P1$ and $P2$ using KDTree to identify reciprocal matches [S2, unofficial]. It builds KDTree for $P1$ and $P2$, queries tree2 for $P1$ to get $nn1\_in\_P2$ (indices of closest points in $P2$ for each point in $P1$), and queries tree1 for $P2$ to get $nn2\_in\_P1$ (indices of closest points in $P1$ for each point in $P2$) using workers=8 [S2, unofficial]. It computes reciprocal matches as $reciprocal\_in\_P1 = (nn2\_in\_P1[nn1\_in\_P2] == np.arange(len(nn1\_in\_P2)))$ and $reciprocal\_in\_P2 = (nn1\_in\_P2[nn2\_in\_P1] == np.arange(len(nn2\_in\_P1)))$, asserting that the sums are equal [S2, unofficial].

### K-Nearest Neighbors Query
K-Nearest Neighbors Query is a method to find the $k$ nearest neighbors for each point in a target set ($new\_xyz$) within a source set ($xyz$), returning indices and squared distances [S3, unofficial]. It performs K-Nearest Neighbors search on 3D points using a CUDA kernel, handling batched inputs with offsets [S3, unofficial]. If $new\_xyz$ is None, it defaults to using $xyz$ as the query set [S3, unofficial]. It calls the CUDA kernel `pointops_cuda.knnquery_cuda` to compute indices ($idx$) and squared distances ($dist2$) [S3, unofficial]. If `transpose` is true, it transposes $x$ and $src$ from $(B, N, 3)$ to $(B, 3, N)$ and makes them contiguous [S3, unofficial]. It extracts batch size $b$, query count $n$, and source count $m$, reshaping $x$ and $src$ to 2D tensors $(B*N, 3)$ and $(B*M, 3)$ [S3, unofficial]. It computes cumulative offsets $x\_offset$ and $src\_offset$ for the batched CUDA kernel call [S3, unofficial]. It calls `knnquery(k, src, x, src_offset, x_offset)` to get indices and distances [S3, unofficial]. It adjusts indices by subtracting $(src\_offset - m)$ to correct for batch offsets, reshapes to $(B, N, K)$, and returns as long integers [S3, unofficial]. Input: $xyz$ $(n, 3)$, $new\_xyz$ $(m, 3)$, offset $(b)$, $new\_offset$ $(b)$. Output: $idx$ $(m, nsample)$, $dist2$ $(m, nsample)$ [S3, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N^2)$ for global kNN (small $N$) and $O(N \cdot C)$ for local kNN (large $N$) [S1].
*   **Memory Complexity**: $O(N)$ for storing points and $O(C)$ for candidate sets [S1].
*   **Measured Performance**: For point counts < 300K, global kNN is used; for higher resolutions/view counts, local kNN is used [S1].
*   **Scaling**: Local kNN scales linearly with $N$ and $C$, making it suitable for large-scale scenes [S1].
*   **Strengths**: Scalability to large point clouds, negligible quality loss compared to global kNN [S1].
*   **Limitations**: Requires camera parameters for local kNN, which may not be available in all scenarios [S1].
*   **When to Use**: Use global kNN for small point clouds (< 300K) and local kNN for large point clouds (>= 300K) [S1].
*   **When Not to Use**: Avoid local kNN if camera parameters are unavailable or if the scene does not have a multi-view grid structure [S1].

## Background
Efficient kNN Implementation originates from the ReSplat project, which focuses on learning recurrent Gaussian splatting [S1]. The method was developed to address the computational bottleneck of global kNN search in large-scale 3D Gaussian splatting scenes [S1]. It builds on the need for efficient neighbor counting in volume rendering, specifically for density evaluation of mesh vertices or Gaussian splats [S1]. The method is cited in [S1] as a key component for scaling Gaussian splatting to high-resolution images.

## Sources
[S1] ReSplat: Learning Recurrent Gaussian Splatting, https://arxiv.org/abs/2510.08575v4, fetched 2026-10-26
[S2] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S3] LogosIntelligenceLab/4d-gaussian-splatting, https://github.com/LogosIntelligenceLab/4d-gaussian-splatting, fetched 2026-10-27 (unofficial)
