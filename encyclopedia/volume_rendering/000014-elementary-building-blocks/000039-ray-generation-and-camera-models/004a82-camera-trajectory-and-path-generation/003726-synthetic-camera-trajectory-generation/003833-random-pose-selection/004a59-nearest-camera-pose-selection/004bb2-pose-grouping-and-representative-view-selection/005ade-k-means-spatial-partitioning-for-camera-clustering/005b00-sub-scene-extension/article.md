# Sub-scene Extension

Also called: Camera division, Cluster expansion, Spatial Decomposition with Conservative Expansion, Spatial partitioning extension

## Overview
Sub-scene extension is a post-processing step applied to initial K-Means clusters to ensure sufficient overlap between neighboring sub-scenes for robust image registration and multi-block rendering [S1]. It sets itself apart from its sibling, [K-Means Spatial Partitioning for Camera Clustering](16768-k-means-spatial-partitioning-for-camera-clustering.md), by actively augmenting cluster radii and adding nearby cameras to prevent registration failures, whereas the parent method only performs static spatial partitioning based on camera center positions [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of initial clusters $C_1, \dots, C_K$ produced by K-Means, where each cluster contains a set of cameras with 3D positions and parameters.
*   **Parameters:**
    *   $\sigma = 1.1$: The expansion threshold (10% expansion) [S1].
    *   $\text{MaxNum}$: The maximum number of cameras the hardware can handle per cluster [S1].
    *   $\text{TopNum}$: The hard upper bound on cameras per cluster after expansion, typically set to $\text{MaxNum}$ or slightly below to ensure safety [S1].
*   **Output:** Expanded clusters $C'_1, \dots, C'_K$ containing augmented camera sets.

**Data Structures:**
Each cluster $C_k$ is a list of camera objects. Each camera object contains:
*   `position`: 3D vector $(x, y, z)$ representing the camera center.
*   `parameters`: Additional camera intrinsics/extrinsics required for rendering.

**Steps:**
1.  **Initialize Initial Size:** Set the target size for initial K-Means clustering to $80\%$ of $\text{MaxNum}$ [S1]. This reserves headroom for the expansion phase [S1].
2.  **Calculate Intra-Cluster Radius:** For each initial cluster $C_k$, calculate the maximum distance from the cluster centroid (or center of mass) to any camera position within the cluster. Let this be $R_k$ [S1].
    *   Formula: $R_k = \max_{c \in C_k} \| \text{position}(c) - \text{centroid}(C_k) \|$
3.  **Expand Radius:** Multiply the radius by the expansion threshold $\sigma$ to get the new radius $R'_k$ [S1].
    *   Formula: $R'_k = R_k \times \sigma$
4.  **Add Neighboring Cameras:** Identify all cameras not currently in $C_k$ that fall within the new radius $R'_k$ from the cluster centroid. Add these cameras to $C_k$ [S1].
5.  **Enforce Hard Cap:** If the number of cameras in the expanded cluster $|C'_k|$ exceeds $\text{TopNum}$, remove cameras from the cluster until $|C'_k| = \text{TopNum}$ [S1]. The specific selection criteria for removal (e.g., furthest from centroid) are not specified in the sources [G].
6.  **Repeat:** Apply steps 2–5 for all initial clusters $K$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sub_scene_extension(clusters, max_num, top_num, sigma=1.1):
    """
    clusters: List of lists, where clusters[k] is a list of camera dicts.
              Each camera dict has 'position' (list of 3 floats).
    max_num: int, hardware limit for initial clustering size.
    top_num: int, hard cap after expansion.
    sigma: float, expansion factor (default 1.1).
    Returns: List of expanded clusters.
    """
    expanded_clusters = []
    
    for k in range(len(clusters)):
        current_cameras = clusters[k]
        if not current_cameras:
            expanded_clusters.append([])
            continue
            
        # Step 2: Calculate centroid
        n = len(current_cameras)
        centroid = [0.0, 0.0, 0.0]
        for cam in current_cameras:
            pos = cam['position']
            centroid[0] += pos[0]
            centroid[1] += pos[1]
            centroid[2] += pos[2]
        centroid[0] /= n
        centroid[1] /= n
        centroid[2] /= n
        
        # Step 2: Calculate max intra-cluster radius R_k
        max_dist_sq = 0.0
        for cam in current_cameras:
            pos = cam['position']
            dx = pos[0] - centroid[0]
            dy = pos[1] - centroid[1]
            dz = pos[2] - centroid[2]
            dist_sq = dx*dx + dy*dy + dz*dz
            if dist_sq > max_dist_sq:
                max_dist_sq = dist_sq
        R_k = max_dist_sq ** 0.5
        
        # Step 3: Expand radius
        R_prime = R_k * sigma
        
        # Step 4: Add neighboring cameras
        # Identify all cameras in the entire scene not already in current_cameras
        # Note: In practice, 'all_cameras' would be a global list passed in or indexed.
        # Here we assume a global list 'all_scene_cameras' is available.
        new_cameras = []
        for cam in all_scene_cameras:
            if cam in current_cameras:
                continue
            pos = cam['position']
            dx = pos[0] - centroid[0]
            dy = pos[1] - centroid[1]
            dz = pos[2] - centroid[2]
            dist_sq = dx*dx + dy*dy + dz*dz
            if dist_sq <= R_prime * R_prime:
                new_cameras.append(cam)
        
        # Combine existing and new cameras
        expanded_cluster = current_cameras + new_cameras
        
        # Step 5: Enforce hard cap TopNum
        if len(expanded_cluster) > top_num:
            # [G] Removal strategy not specified in source. 
            # Common practice is to remove cameras furthest from centroid.
            # Sorting by distance descending and keeping top_num.
            expanded_cluster.sort(key=lambda c: 
                (c['position'][0]-centroid[0])**2 + 
                (c['position'][1]-centroid[1])**2 + 
                (c['position'][2]-centroid[2])**2, 
                reverse=True)
            expanded_cluster = expanded_cluster[:top_num]
            
        expanded_clusters.append(expanded_cluster)
        
    return expanded_clusters
```

**Edge Cases and Pitfalls:**
*   **GPU Memory Overflow:** Expansion may cause the number of cameras to exceed the initial $\text{MaxNum}$ limit [S1]. This is handled by enforcing the hard upper bound $\text{TopNum}$ [S1].
*   **Degenerate Input:** If a cluster has only one camera, the radius is 0. Expansion will still create a non-zero radius based on $\sigma$, potentially adding neighbors.

## Operating characteristics
*   **Time Complexity:** $O(K \cdot N)$, where $K$ is the number of clusters and $N$ is the total number of cameras in the scene, as each cluster expansion requires checking distances to all other cameras [S1].
*   **Memory Complexity:** Increases memory usage per cluster due to added overlapping cameras, but bounded by $\text{TopNum}$ [S1].
*   **Strengths:** Enhances the success rate of image registration and multi-block rendering by ensuring spatial overlap [S1]. Balances memory utilization and hardware reliability [S1].
*   **Limitations:** Requires careful tuning of $\text{TopNum}$ to avoid memory overflow [S1]. The specific algorithm for removing cameras when capping is not defined [G].
*   **When to use:** When using K-Means for large-scale scene partitioning and subsequent image registration is required [S1].
*   **When not to use:** If GPU memory is extremely constrained and overlap is not critical for the downstream task.

## Background
*   **Origin:** Introduced in BirdNeRF for fast neural reconstruction of large-scale scenes from aerial imagery [S1].
*   **History:** Developed to address failures in image registration caused by insufficient overlap between spatially partitioned sub-scenes [S1].
*   **Usage:** Used in aerial imagery reconstruction pipelines where multiple blocks are rendered and aligned [S1].

## Sources
[S1] BirdNeRF: fast neural reconstruction of large-scale scenes from aerial imagery., https://doi.org/10.1038/s41598-025-21206-z, fetched 2026-10-26
