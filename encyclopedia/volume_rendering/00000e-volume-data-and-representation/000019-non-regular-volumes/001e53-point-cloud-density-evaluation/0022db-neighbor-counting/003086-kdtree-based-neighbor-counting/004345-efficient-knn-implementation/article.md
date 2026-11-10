# Efficient kNN Implementation

Also called: Customized CUDA kNN, Local kNN with Camera Parameters

## Overview
Efficient kNN Implementation provides scalable nearest-neighbor search for point cloud density evaluation in volume rendering, specifically addressing the computational bottleneck of global $O(N^2)$ searches at high resolutions [S1]. It distinguishes itself by offering two modes: a high-performance customized CUDA global kNN for moderate point counts (<300K) and an $O(N \cdot C)$ efficient local kNN that leverages camera parameters to restrict candidate sets for massive point clouds [S1].

## How it works
The method selects an implementation strategy based on the total number of points $N$ and the number of views.

**Inputs:**
- Point cloud positions $P \in \mathbb{R}^{N \times 3}$.
- Camera parameters (intrinsics and extrinsics) for all views.
- Target neighbor count $k$.
- Threshold $N_{thresh} = 300,000$ [S1].

**Outputs:**
- For each point, the indices of its $k$ nearest neighbors.

**Data Structures:**
- **Global Mode:** Standard CUDA device arrays for points and distances.
- **Local Mode:**
  - Candidate set $C$: A subset of points identified as spatial neighbors in the source view and cross-view neighbors projected from nearby cameras [S1].
  - Size of $C$ is constant $C$ per query [S1].

**Algorithm Steps:**

1. **Mode Selection:**
   - If $N < N_{thresh}$, execute **Customized CUDA Global kNN**.
   - If $N \ge N_{thresh}$, execute **Efficient Local kNN**.

2. **Customized CUDA Global kNN (for $N < 300,000$):**
   - Perform a brute-force distance calculation between all pairs or use optimized GPU kernels to find the $k$ nearest neighbors for each point [S1].
   - Complexity is $O(N^2)$ [S1].

3. **Efficient Local kNN (for $N \ge 300,000$):**
   - **Candidate Construction:** For each query point, construct a candidate set of size $C$ [S1]:
     - Identify spatial neighbors within the same source view.
     - Identify cross-view neighbors by projecting points from nearby cameras into the current view's coordinate space [S1].
   - **Distance Computation:** Compute exact 3D Euclidean distances between the query point and all points in the candidate set $C$ [S1].
   - **Selection:** Select the top-$k$ points from the candidate set based on the computed distances [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def efficient_knn(points, camera_params, k, n_thresh=300000):
    """
    points: List of 3D coordinates (N, 3)
    camera_params: Dictionary or list of camera intrinsics/extrinsics
    k: Number of neighbors to find
    n_thresh: Threshold for switching modes
    """
    n = len(points)
    
    if n < n_thresh:
        # Customized CUDA Global kNN
        # [S1] Uses a customized CUDA implementation
        # [G] Assuming a standard CUDA kernel call or library function
        # that computes pairwise distances and selects top-k.
        neighbors = cuda_global_knn(points, k)
        return neighbors
    else:
        # Efficient Local kNN with O(N*C) complexity
        # [S1] Restricts search to candidate set of size C
        neighbors = []
        for i, point in enumerate(points):
            # Construct candidate set C
            # [S1] Aggregate spatial neighbors from same view
            # [S1] Aggregate cross-view neighbors projected from nearby cameras
            candidates = construct_candidate_set(point, camera_params, i)
            
            # Compute exact 3D distances within this set
            # [S1]
            dists = []
            for c_idx, c_point in enumerate(candidates):
                d = euclidean_distance(point, c_point)
                dists.append((d, c_idx))
            
            # Sort and select top-k
            # [S1]
            dists.sort(key=lambda x: x[0])
            top_k_indices = [d[1] for d in dists[:k]]
            
            # Map local candidate indices back to global point indices
            # [G] Assuming candidates stores global indices
            global_neighbors = [candidates[idx] for idx in top_k_indices]
            neighbors.append(global_neighbors)
            
        return neighbors

def construct_candidate_set(query_point, camera_params, query_idx):
    """
    Constructs the candidate set C for a query point.
    [S1] Uses camera parameters to find spatial and cross-view neighbors.
    """
    candidates = []
    
    # [S1] Spatial neighbors from same view
    # [G] Assuming view_id maps points to views
    view_id = get_view_id(query_idx)
    same_view_points = get_points_in_view(view_id)
    for p_idx, p in enumerate(same_view_points):
        if p_idx != query_idx: # Avoid self-match if necessary
            candidates.append(p)
            
    # [S1] Cross-view neighbors projected from nearby cameras
    nearby_cameras = get_nearby_cameras(camera_params, query_point)
    for cam in nearby_cameras:
        # Project points from other views into current view space
        # [S1]
        projected_points = project_points_from_camera(cam, camera_params)
        for p in projected_points:
            candidates.append(p)
            
    return candidates

def euclidean_distance(p1, p2):
    return ((p1[0]-p2[0])**2 + (p1[1]-p2[1])**2 + (p1[2]-p2[2])**2)**0.5
```

**Parameters:**
- $N_{thresh}$: 300,000 points [S1].
- $C$: Size of candidate set, constant per query [S1]. Value not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** Exact 3D distances are computed within the candidate set to guarantee correctness [S1].
- **Boundaries:** The candidate construction relies on accurate camera parameters; errors in projection may lead to missing true neighbors [S1].
- **Degenerate Input:** If $N$ is very large but views are sparse, the candidate set construction must ensure $C$ remains small to maintain $O(N \cdot C)$ efficiency [S1].

## Subtopics

### [Efficient Local kNN](10975-efficient-local-knn.md)
Efficient Local kNN is a drop-in replacement for global brute-force k-nearest neighbors search that exploits the multi-view grid structure of point clouds to reduce computational complexity from $O(N^2)$ to $O(N)$. It differs from global kNN by constraining the candidate search space to spatially adjacent pixels in the source view and corresponding projected pixels in other views, rather than scanning all $N$ points. This method is designed for high-resolution point cloud representations, such as Gaussian splatting, where standard kNN attention becomes a bottleneck.

### Reciprocal Nearest Neighbor Matching
Reciprocal Nearest Neighbor Matching finds mutual nearest neighbors between two point sets $P1$ and $P2$ using KDTree to identify reciprocal matches [S2, unofficial]. It builds KDTree for $P1$ and $P2$, queries tree2 for $P1$ to get $nn1\_in\_P2$ (indices of closest points in $P2$ for each point in $P1$), and queries tree1 for $P2$ to get $nn2\_in\_P1$ (indices of closest points in $P1$ for each point in $P2$) using workers=8 [S2, unofficial]. It computes $reciprocal\_in\_P1 = (nn2\_in\_P1[nn1\_in\_P2] == np.arange(len(nn1\_in\_P2)))$ and $reciprocal\_in\_P2 = (nn1\_in\_P2[nn2\_in\_P1] == np.arange(len(nn2\_in\_P1)))$, asserting sums are equal, and returns $reciprocal\_in\_P2, nn2\_in\_P1, sum$ [S2, unofficial].

## Operating characteristics
- **Time Complexity:**
  - Global Mode: $O(N^2)$ [S1].
  - Local Mode: $O(N \cdot C)$ [S1].
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** The customized CUDA implementation is efficient for point counts < 300K [S1]. The local kNN alternative ensures scalability for higher resolutions or view counts where the global implementation becomes a bottleneck [S1].
- **Strengths:** Scalability to high-resolution images and large point clouds [S1].
- **Limitations:** Global mode is limited to < 300K points [S1]. Local mode depends on the quality of camera parameters and candidate set construction [S1].
- **When to Use:** Use global mode for small to medium point clouds (< 300K) [S1]. Use local mode for large point clouds or high-resolution views [S1].

## Background
The method originates from ReSplat: Learning Recurrent Gaussian Splatting [S1]. It was developed to address the computational bottleneck of global kNN in Gaussian splatting applications [S1]. The reciprocal nearest neighbor matching variant is associated with NVlabs/InstantSplat [S2, unofficial].

## Sources
[S1] ReSplat: Learning Recurrent Gaussian Splatting, https://arxiv.org/abs/2510.08575v4, fetched 2026-10-26
[S2] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
