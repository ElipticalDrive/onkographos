# Efficient Local kNN

Also called: Local k-nearest neighbors, O(N) local kNN

## Overview
Efficient Local kNN is a drop-in replacement for global brute-force k-nearest neighbors (kNN) search in point cloud or Gaussian splatting rendering pipelines [S1]. It solves the $O(N^2)$ complexity bottleneck of global search by exploiting the multi-view grid structure of the scene, reducing complexity to $O(N)$ [S1]. It sets itself apart from global kNN by constraining the search space to spatially adjacent pixels in the source view and corresponding projected pixels in other views, achieving significant speedups with negligible quality loss [S1].

## How it works
**Inputs:**
- A set of $N$ 3D points (e.g., Gaussian centers) with positions $\mathbf{p}_i \in \mathbb{R}^3$ [S1].
- $V$ camera views, each defined by an intrinsic matrix $\mathbf{K}_v$ and extrinsic pose $\mathbf{T}_v$ (world-to-camera transformation) [G].
- Integer parameters: $k$ (number of neighbors), $r_s$ (same-view spatial radius), $r_c$ (cross-view spatial radius) [S1].
- Optional: Depth maps or point cloud projections for efficient spatial indexing [S1].

**Outputs:**
- For each query point $\mathbf{p}_q$, a list of $k$ nearest neighbor indices from the candidate set [S1].

**Data Structures:**
- **Point Cloud:** Array of $N$ points, each storing 3D coordinates [S1].
- **Camera Info:** Array of $V$ cameras, each storing $\mathbf{K}_v$ and $\mathbf{T}_v$ [G].
- **Spatial Index (Implicit):** The method relies on the assumption that nearby pixels in 2D image space correspond to nearby 3D points [S1]. Candidates are identified by 2D pixel coordinates within a square window [S1].

**Parameters:**
- $r_s = 3$: Spatial radius for the same-view window [S1].
- $r_c = 3$: Spatial radius for the cross-view window [S1].
- $k$: Number of neighbors to retrieve [S1].
- Compression Factor: Default 16×, using depth maps at 1/4 resolution [S1].

**Steps:**
1. **Same-View Candidate Generation:** For each query point $\mathbf{p}_q$ in the current view, project it to 2D pixel coordinates $(u, v)$ [S1]. Define a square window of size $(2r_s + 1) \times (2r_s + 1)$ centered at $(u, v)$ [S1]. Collect all 3D points whose 2D projections fall within this window in the same view [S1].
2. **Cross-View Candidate Gathering:** For each other view $v' \neq v$:
   a. Project the 3D query point $\mathbf{p}_q$ into the 2D image plane of view $v'$ using $\mathbf{K}_{v'}$ and $\mathbf{T}_{v'}$ [S1]. Let the projected pixel be $(u', v')$ [S1].
   b. Define a square window of size $(2r_c + 1) \times (2r_c + 1)$ centered at $(u', v')$ [S1].
   c. Collect all 3D points from view $v'$ whose 2D projections fall within this window [S1].
3. **Candidate Aggregation:** Combine the same-view candidates and all cross-view candidates into a single candidate set $\mathcal{C}$ [S1].
4. **Neighbor Selection:** Compute the 3D Euclidean distance between $\mathbf{p}_q$ and every point in $\mathcal{C}$ [S1]. Select the $k$ points with the smallest distances [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def efficient_local_knn(points, cameras, k, rs=3, rc=3):
    """
    Efficient Local kNN implementation.
    
    Args:
        points: Nx3 array of 3D point coordinates.
        cameras: List of V dictionaries, each containing 'K' (3x3 intrinsic) 
                 and 'T' (4x4 extrinsic world-to-camera matrix).
        k: Number of neighbors to find.
        rs: Same-view spatial radius.
        rc: Cross-view spatial radius.
        
    Returns:
        neighbors: Nxk array of neighbor indices.
    """
    N = points.shape[0]
    V = len(cameras)
    neighbors = np.zeros((N, k), dtype=np.int32)
    
    # Precompute 2D projections for all points in all views.
    # [S1] The sources do not specify the exact spatial index structure used for O(1) window queries.
    # This implementation uses direct projection and filtering for correctness.
    proj_2d = np.zeros((N, V, 2))
    for v_idx in range(V):
        K = cameras[v_idx]['K']
        T = cameras[v_idx]['T']
        points_h = np.hstack([points, np.ones((N, 1))])
        points_cam = (T @ points_h.T).T[:, :3]
        points_2d_h = (K @ points_cam.T).T
        proj_2d[:, v_idx, 0] = points_2d_h[:, 0] / points_2d_h[:, 2]
        proj_2d[:, v_idx, 1] = points_2d_h[:, 1] / points_2d_h[:, 2]

    for i in range(N):
        query_point = points[i]
        candidates = []
        
        # Step 1: Same-view candidates (View 0 is assumed as the source view)
        u, v = proj_2d[i, 0]
        mask_s = (np.abs(proj_2d[:, 0, 0] - u) <= rs) & (np.abs(proj_2d[:, 0, 1] - v) <= rs)
        same_view_indices = np.where(mask_s)[0]
        candidates.extend(same_view_indices)
        
        # Step 2: Cross-view candidates
        for v_idx in range(1, V):
            u_cross, v_cross = proj_2d[i, v_idx]
            mask_c = (np.abs(proj_2d[:, v_idx, 0] - u_cross) <= rc) & (np.abs(proj_2d[:, v_idx, 1] - v_cross) <= rc)
            cross_view_indices = np.where(mask_c)[0]
            candidates.extend(cross_view_indices)
        
        # Step 3 & 4: Select top-k neighbors
        if len(candidates) == 0:
            # [G] Fallback if no candidates found
            neighbors[i, :] = i
            continue
            
        candidate_points = points[candidates]
        distances = np.linalg.norm(candidate_points - query_point, axis=1)
        
        k_actual = min(k, len(candidates))
        top_k_indices = np.argpartition(distances, k_actual)[:k_actual]
        neighbors[i, :k_actual] = np.array(candidates)[top_k_indices]
        neighbors[i, k_actual:] = i
        
    return neighbors
```

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point precision in projection calculations may affect window boundary checks [G].
- **Degenerate Input:** If $N$ is small or points are sparse, the candidate set may be empty [G].
- **Compression:** The sources state that 16× compression using 1/4 resolution depth maps provides a good speed-accuracy trade-off [S1].

## Subtopics

### Morton-order approximation for KNN/LD scoring
Replaces exact kNN distance computation with a Morton-order (Z-order curve) spatial approximation to compute Local Distinctiveness (LD) scores for Gaussian pruning [S2]. It sorts all Gaussians by their Morton code index and uses the resulting spatial ordering to approximate nearest-neighbor relationships for LD scoring instead of computing pairwise Euclidean distances [S2]. This achieves a 146× speedup over exact KNN while maintaining nearly identical accuracy, making it suitable for scalability in complex or large-scale scenes where KNN's $O(N^2)$ complexity becomes impractical [S2]. On the Mip-NeRF 360 dataset (OMG-XS variant), Morton order yields PSNR 27.06, SSIM 0.807, LPIPS 0.243, and Time 4100 ms, compared to KNN's PSNR 27.07, SSIM 0.808, LPIPS 0.241, and Time 28 ms [S2].

## Operating characteristics
**Time and Memory Complexity:**
- Reduces complexity from $O(N^2)$ for global brute-force search to $O(N)$ by vectorizing cross-view reprojections and limiting the search space [S1].
- Morton-order approximation avoids the $O(N^2)$ time complexity and high memory usage of exact KNN, achieving a 146× speedup [S2].

**Measured Performance:**
- On 8 views at 512x960 resolution with 246K Gaussians, Local kNN takes 0.591s compared to Global kNN's 0.816s [S1].
- Morton-order approximation achieves a 146× speedup over KNN [S2].

**Scaling:**
- Local kNN scales linearly with the number of points $N$ [S1].
- Morton-order approximation scales better with the number of primitives, making it suitable for large-scale scenes [S2].

**Strengths and Limitations:**
- **Strengths:** Significant speedup with negligible quality loss (PSNR 27.65 vs 27.70 for Local kNN) [S1]. Morton order provides scalability and generalizability [S2].
- **Limitations:** Relies on the assumption that nearby pixels in 2D correspond to nearby 3D points, which may not hold for all scene geometries [S1].

**When to Use:**
- Use Local kNN when inference speed is critical and the scene has a multi-view grid structure [S1].
- Use Morton-order approximation when the number of Gaussians is large and KNN becomes computationally prohibitive [S2].

## Background
Efficient Local kNN was introduced in ReSplat: Learning Recurrent Gaussian Splatting [S1]. It addresses the computational bottleneck of global kNN search in high-resolution point clouds [S1]. Morton-order approximation for KNN/LD scoring was introduced in Optimized Minimal 3D Gaussian Splatting [S2] to improve scalability during training [S2].

## Sources
[S1] ReSplat: Learning Recurrent Gaussian Splatting, https://arxiv.org/abs/2510.08575v4, fetched 2026-10-26
[S2] Optimized Minimal 3D Gaussian Splatting, https://doi.org/10.52202/085713-4534, fetched 2026-10-30
