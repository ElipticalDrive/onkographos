# Point Cloud Regularization

Also called: Multi-view Reliability Filtering and K-means Regularization

## Overview
Point Cloud Regularization is a post-processing pipeline applied to an initial colored point cloud to remove noise, duplicates, and outliers before training a 3D Gaussian Splatting (3DGS) model [S1]. It solves the problem of unreliable initialization caused by merging Structure-from-Motion (SfM) points with lightweight 3DGS primitive centers, which introduces depth ambiguity, duplicated points from densification, and unstable tracks [S1]. It sets itself apart from its sibling [Alpha-based Point Cleaning](19085-alpha-based-point-cleaning.md) by explicitly enforcing multi-view consistency and geometric normal agreement rather than relying solely on alpha-blending reliability metrics [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An initial colored point cloud $P_{init} = \{(X_k, C_k)\}$, formed by the union of SfM points $P_0$ and lightweight 3DGS primitive centers $P_1$ [S1]. Each point $X_k$ has a 3D position and color $C_k$ [S1].
*   **Output:** A regularized point cloud $P_{final}$ containing only geometrically consistent and reliable points [S1].

**Data Structures:**
*   $P_{init}$: Set of points with position $X \in \mathbb{R}^3$ and color $C$.
*   $P_{sv}$: Subset of $P_{init}$ supported by a single camera view.
*   $P_{mv}$: Subset of $P_{init}$ supported by multiple camera views.
*   $P_{filter}$: Intermediate point cloud after single-view filtering.
*   $P_{clu}$: Intermediate point cloud after clustering-based denoising.
*   $N_k$: Set of 10 nearest neighbors of point $X_k$ in Euclidean space.

**Parameters:**
*   $K = 1000$: Number of clusters for K-means [S1].
*   $\alpha = 0.20$: Fraction of single-view points to retain (top 20%) [S1].
*   $\beta = 0.30$: Fraction of points to retain within each cluster (nearest 30%) [S1].
*   $\tau_{cos} = 0.2$: Threshold for mean cosine similarity of normals [S1].
*   $k_{nn} = 10$: Number of neighbors for normal estimation [S1].

**Steps:**

1.  **Single-view Point Filtering:**
    Split $P_{init}$ into single-view supported points $P_{sv}$ and multi-view supported points $P_{mv}$ based on camera projection [S1]. For each point $X \in P_{sv}$, compute a reliability score $r(X)$ based on its proximity to $P_{mv}$ [S1]. Retain the top 20% of points from $P_{sv}$ ranked by $r(X)$, denoted as $P_{sv}^*$ [S1]. Create the filtered point cloud $P_{filter} = P_{mv} \cup P_{sv}^*$ [S1].

2.  **Clustering-Based Denoising:**
    Apply K-means clustering with $K=1000$ clusters to $P_{filter}$ [S1]. For each cluster $c$ with centroid $\mu_c$ and point set $Q_c$, retain the nearest 30% of points to $\mu_c$ [S1]. The cluster-filtered cloud is defined as:
    $$ P_{clu} = \{ X_k \in P_{filter} \mid k \in \text{Top30\%} \{ \|X_k - \mu_c\|^2 : X_k \in Q_c \}, c = 1, \dots, K \} $$
    [S1].

3.  **Normal-based Consistency Filtering:**
    For each point $X_k \in P_{clu}$, estimate its surface normal $n_k$ via Principal Component Analysis (PCA) on its 10 nearest neighbors $N_k$ in Euclidean space [S1]. Let all normals be unit-length. Compute the mean cosine similarity:
    $$ \bar{c}_k = \frac{1}{|N_k|} \sum_{j \in N_k} n_k^\top n_j $$
    [S1]. Retain point $k$ if $\bar{c}_k \ge 0.2$ [S1]. The final set is:
    $$ P_{final} = \{ (X_k, C_k) \in P_{clu} \mid \bar{c}_k \ge 0.2 \} $$
    [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def point_cloud_regularization(P_init, camera_projections, K=1000, alpha=0.20, beta=0.30, tau_cos=0.2, k_nn=10):
    """
    Regularizes an initial point cloud by removing noise and unreliable points.
    
    Args:
        P_init: List of points, each with position X and color C.
        camera_projections: Mapping of point index to list of camera indices observing it.
        K: Number of clusters for K-means.
        alpha: Fraction of single-view points to retain.
        beta: Fraction of points to retain per cluster.
        tau_cos: Minimum mean cosine similarity for normal consistency.
        k_nn: Number of neighbors for normal estimation.
        
    Returns:
        P_final: List of regularized points.
    """
    # Step 1: Single-view Point Filtering
    P_sv = []
    P_mv = []
    
    for i, point in enumerate(P_init):
        cameras = camera_projections[i]
        if len(cameras) == 1:
            P_sv.append(point)
        else:
            P_mv.append(point)
            
    # Compute reliability scores for single-view points based on proximity to P_mv
    # Note: Exact formula for r(X) is not provided in sources, but implies proximity metric [S1]
    # [G] Assuming r(X) is inverse distance to nearest neighbor in P_mv
    import math
    from collections import defaultdict
    
    # Precompute distances from P_sv to P_mv for reliability scoring
    # This is a placeholder for the specific proximity metric r(X) defined in [S1]
    # The source states "based on its proximity to Pmv" but does not give the explicit scalar formula.
    # We assume a standard inverse Euclidean distance or similar reliability metric.
    # To be implementable, we define r(X) as the inverse distance to the nearest point in P_mv.
    
    if not P_mv:
        # If no multi-view points exist, all single-view points are unreliable or undefined.
        # [G] Fallback: retain none or all? Source implies P_mv exists for this step.
        # If P_mv is empty, P_sv* is empty.
        P_sv_star = []
    else:
        # Calculate reliability scores
        reliability_scores = []
        for point_sv in P_sv:
            min_dist = float('inf')
            for point_mv in P_mv:
                dist = math.sqrt(sum((a-b)**2 for a, b in zip(point_sv.X, point_mv.X)))
                if dist < min_dist:
                    min_dist = dist
            # Reliability is higher for points closer to multi-view points
            # [G] Using inverse distance as proxy for r(X) as exact formula not given
            r = 1.0 / (min_dist + 1e-8) 
            reliability_scores.append((point_sv, r))
            
        # Sort by reliability descending
        reliability_scores.sort(key=lambda x: x[1], reverse=True)
        
        # Retain top alpha fraction
        num_retain = max(1, int(len(reliability_scores) * alpha))
        P_sv_star = [p for p, r in reliability_scores[:num_retain]]
        
    P_filter = P_mv + P_sv_star
    
    # Step 2: Clustering-Based Denoising
    # Apply K-means clustering
    # [G] Standard K-means implementation required. 
    # We assume a function kmeans(points, K) that returns (centroids, labels)
    centroids, labels = kmeans([p.X for p in P_filter], K)
    
    # Group points by cluster
    clusters = defaultdict(list)
    for i, point in enumerate(P_filter):
        clusters[labels[i]].append(point)
        
    P_clu = []
    for c_idx, points_in_cluster in clusters.items():
        centroid = centroids[c_idx]
        # Calculate distances to centroid
        dists = []
        for point in points_in_cluster:
            dist = sum((a-b)**2 for a, b in zip(point.X, centroid))
            dists.append((point, dist))
            
        # Sort by distance ascending
        dists.sort(key=lambda x: x[1])
        
        # Retain top beta fraction (nearest points)
        num_retain_cluster = max(1, int(len(dists) * beta))
        retained = [p for p, d in dists[:num_retain_cluster]]
        P_clu.extend(retained)
        
    # Step 3: Normal-based Consistency Filtering
    P_final = []
    
    # Precompute neighbor indices for efficiency if needed, but for clarity:
    for point in P_clu:
        # Find k_nn nearest neighbors in Euclidean space
        neighbors = get_k_nearest_neighbors(point, P_clu, k_nn)
        
        # Estimate normal via PCA on neighbors
        # [G] PCA on covariance matrix of neighbor positions relative to point
        neighbor_positions = [n.X for n in neighbors]
        n_k = estimate_normal_pca(neighbor_positions, point.X)
        
        # Compute mean cosine similarity
        cos_sims = []
        for neighbor in neighbors:
            # Estimate neighbor normal similarly or use precomputed?
            # Source says "angular deviation of n(X) from its neighbors"
            # We need normals for neighbors. For simplicity in self-contained code:
            # [G] Re-estimate normal for each neighbor using its own neighbors
            # This is computationally expensive but correct per description.
            # Optimization: Precompute all normals.
            n_j = estimate_normal_pca([n.X for n in get_k_nearest_neighbors(neighbor, P_clu, k_nn)], neighbor.X)
            # Dot product for cosine similarity (unit vectors)
            cos_sim = sum(a*b for a, b in zip(n_k, n_j))
            cos_sims.append(cos_sim)
            
        mean_cos_sim = sum(cos_sims) / len(cos_sims)
        
        if mean_cos_sim >= tau_cos:
            P_final.append(point)
            
    return P_final

def get_k_nearest_neighbors(query_point, point_cloud, k):
    # Returns list of k points from point_cloud closest to query_point.X
    # Excludes query_point itself if present
    dists = []
    for p in point_cloud:
        if p is query_point:
            continue
        d = sum((a-b)**2 for a, b in zip(p.X, query_point.X))
        dists.append((p, d))
    dists.sort(key=lambda x: x[1])
    return [p for p, d in dists[:k]]

def estimate_normal_pca(neighbor_positions, center_pos):
    # Compute covariance matrix of neighbor positions relative to center
    # Then find eigenvector with smallest eigenvalue
    # [G] Simplified PCA normal estimation
    import numpy as np
    points = np.array(neighbor_positions)
    mean = np.mean(points, axis=0)
    diff = points - mean
    cov = np.dot(diff.T, diff)
    eigenvalues, eigenvectors = np.linalg.eigh(cov)
    # Normal is eigenvector corresponding to smallest eigenvalue
    normal = eigenvectors[:, 0]
    # Normalize
    norm = np.linalg.norm(normal)
    if norm > 0:
        normal = normal / norm
    return normal.tolist()

def kmeans(points, K):
    # [G] Standard K-means implementation omitted for brevity, 
    # but required for implementation. Returns centroids and labels.
    pass
```

**Edge Cases and Pitfalls:**
*   **Empty Multi-view Set:** If $P_{mv}$ is empty, the reliability score $r(X)$ cannot be computed as defined. The pseudocode handles this by returning an empty $P_{sv}^*$ [S1].
*   **Small Clusters:** If a cluster has fewer than $1/\beta$ points, the "nearest 30%" logic may retain all or none depending on rounding. The pseudocode uses `max(1, ...)` to ensure at least one point is retained if possible, but the source does not specify this guard [G].
*   **Normal Estimation Degeneracy:** PCA on collinear or coplanar neighbors may yield unstable normals. The source does not specify a fallback [G].
*   **Precision:** The threshold $\bar{c}_k \ge 0.2$ is specific to the source [S1].

## Operating characteristics
*   **Time Complexity:** Dominated by K-means clustering ($O(N \cdot K \cdot I)$ where $I$ is iterations) and normal estimation ($O(N \cdot k_{nn} \cdot \log N)$ or $O(N \cdot k_{nn})$ with spatial indexing) [S1].
*   **Memory Complexity:** $O(N)$ to store the point cloud and intermediate subsets [S1].
*   **Measured Performance:** On MipNeRF360 and LLFF datasets, regularization increases computation time (e.g., from 1m32s to 4m16s training time in some configurations) but mitigates overfitting and improves reconstruction quality [S1].
*   **Strengths:** Removes noise from unstable 2-view SfM tracks and duplicated points from 3DGS densification [S1]. Enforces geometric consistency [S1].
*   **Limitations:** Computationally expensive compared to vanilla 3DGS [S1]. Initialization strength is more decisive than regularization gains [S1].
*   **When to use:** When initializing 3DGS with a merged point cloud from SfM and lightweight 3DGS, especially in sparse-view scenarios [S1].
*   **When not to use:** When computational resources are severely limited and initialization quality is already high [S1].

## Background
Point Cloud Regularization was introduced in the paper "Initialize to Generalize: A Stronger Initialization Pipeline for Sparse-View 3DGS" by CoR-GS authors [S1]. It addresses the initialization-limited nature of sparse-view 3DGS by refining the merged point cloud $P_{init}$ before final optimization [S1]. The method combines single-view filtering, clustering-based denoising, and normal-based consistency filtering to ensure uniform spatial coverage and multi-view consistency [S1].

## Sources
[S1] Initialize to Generalize: A Stronger Initialization Pipeline for Sparse-View 3DGS., https://doi.org/10.1109/tvcg.2026.3736471, fetched 2026-10-26
