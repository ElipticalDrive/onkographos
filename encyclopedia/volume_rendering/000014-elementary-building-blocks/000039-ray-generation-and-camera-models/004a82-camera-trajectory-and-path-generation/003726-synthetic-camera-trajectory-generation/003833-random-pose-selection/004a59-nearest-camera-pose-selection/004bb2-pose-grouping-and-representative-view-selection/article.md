# Pose Grouping and Representative View Selection

Also called: AABB Expansion, Block Splitting, Clustering-based View Selection, KMeans Block Splitting, Representative View Selection

## Overview
This method selects a subset of representative training viewpoints from a large set of rendered images by clustering camera poses and filtering for image quality [S1]. It addresses the problem of visual inconsistencies, amplified exposure differences, and noise propagation that arise when using all available frames for subsequent training rounds [S1]. Unlike its parent [Nearest Camera Pose Selection](12726-nearest-camera-pose-selection.md), which selects the single most relevant image based on minimal distance, this method partitions the entire dataset into groups to ensure geometric coverage and selects the sharpest image from each group [S1]. It differs from SfM-based splitting methods (e.g., Mega-NeRF) by leveraging known camera pose priors and KMeans clustering on centers rather than feature-metric correspondences, avoiding failures in texture-less regions [S2, unofficial].

## How it works
**Inputs:**
- Set of $N$ camera poses $P = \{P_1, \dots, P_N\}$, where each $P_i$ consists of rotation matrix $R_i$ and translation vector $t_i$ [S1].
- Set of rendered images $I = \{I_1, \dots, I_N\}$ corresponding to the poses [S1].
- Number of clusters $K$ (typically 12) [S1].
- Bounding box scale factor vector $s_{AABB} = [s_x, s_y, s_z]^T$ (default $s_x=s_y=s_z=1.0$) [S2, unofficial].

**Outputs:**
- A set of $K$ representative images, one from each cluster/block [S1].

**Data Structures:**
- Camera Center: $c_n = -R_n^T t_n$ [S1].
- Forward Direction: $z_n = R_n^T [0,0,1]^T$ [S1].
- Feature Vector: $v_n = [c_n, z_n] \in \mathbb{R}^6$ [S1].
- Axis-Aligned Bounding Box (AABB): Defined by minimum corner $A$ and maximum corner $B$ [S2, unofficial].

**Steps:**

1. **Extract Geometric Features:**
   For each image $i$, compute the camera center $c_i$ and forward direction $z_i$ from the extrinsic parameters $R_i, t_i$ [S1]. Construct the 6D feature vector $v_i = [c_i, z_i]$ [S1].

2. **Initial Clustering:**
   Perform KMeans clustering on the set of feature vectors $\{v_i\}$ with $K$ clusters [S1]. Assign each image $i$ to an initial block based on its cluster ID [S2, unofficial].
   $$ LI_i = \text{KMeans}(v_i, K) $$
   where $LI_i$ is the block ID of image $i$ [S2, unofficial].

3. **Compute Bounding Boxes:**
   For each cluster $k$, determine the spatial extent of the camera centers $P_k = \{P_i \mid LI_i = k\}$ [S2, unofficial]. Compute the initial AABB bounds $A_k$ and $B_k$:
   $$ A_k = \min(P_k), \quad B_k = \max(P_k) $$
   where min/max are taken component-wise over the 3D coordinates of the camera centers in the cluster [S2, unofficial].

4. **Expand Bounding Boxes (AABB Expansion):**
   To introduce overlap between blocks, expand the AABBs using the scale factor $s_{AABB}$ [S2, unofficial].
   - Compute the diagonal midpoint $C_k$:
     $$ C_k = \frac{A_k + B_k}{2} $$
   - Compute the diagonal direction $r_k$ and half-length $len_k$:
     $$ r_k = \frac{A_k - B_k}{\|A_k - B_k\|}, \quad len_k = \frac{\|B_k - A_k\|}{2} $$
   - Recompute the expanded bounds $A'_k$ and $B'_k$:
     $$ A'_k = C_k + s_{AABB} \cdot r_k \cdot len_k $$
     $$ B'_k = C_k - s_{AABB} \cdot r_k \cdot len_k $$
   Note: The source notation uses $A$ and $B$ for the final bounds after expansion [S2, unofficial].

5. **Regroup Images:**
   Reassign images to blocks based on the expanded bounds [S2, unofficial]. If the camera pose $P_i$ of image $i$ lies within the expanded bounds $[A'_k, B'_k]$ (component-wise), add image $i$ to block $B_k$ [S2, unofficial]. This allows images to belong to multiple overlapping blocks [S2, unofficial].
   $$ \text{If } A'_k \le P_i \le B'_k, \text{ then } B_k \leftarrow B_k \cup \{I_i\} $$

6. **Select Representative Views:**
   For each block $B_k$, calculate the Laplacian variance for every image $I_i \in B_k$ [S1]. Select the image with the highest variance as the representative view for that block [S1].
   $$ \text{Representative}_k = \arg \max_{I_i \in B_k} \text{Var}_{Lap}(I_i) $$

**Formula:**
The sharpness metric is the variance of the Laplacian gradient:
$$ \text{Var}_{Lap}(I) = \text{Var}(\nabla^2 I) $$
where $\nabla^2 I$ is the Laplacian gradient of image $I$ [S1]. Higher values indicate richer high-frequency details (texture/edges) [S1].

**Parameters:**
- $K$: Number of clusters/blocks. Default 12 [S1].
- $s_{AABB}$: Scale factor vector. Default $[1, 1, 1]^T$ (no overlap) [S2, unofficial].
- $\gamma$: Hyperparameter for Inverse Distance Weighting (used in blending, not selection). Default 1 [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Texture-less Regions:** SfM-based splitting fails here; this method uses pose priors, making it robust [S2, unofficial].
- **Overlap Ratio:** If $s_{AABB} \to \infty$, blocks encompass all training images [S2, unofficial]. If $s_{AABB} = 1.0$, there is no overlap [S2, unofficial].
- **Blurring:** If blocks are too close, blending methods like Inverse Distance Weighting (IDW) can produce blurry images due to similar weights for outliers [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def pose_grouping_and_representative_view_selection(poses, images, K=12, s_AABB=None):
    if s_AABB is None:
        s_AABB = [1.0, 1.0, 1.0]
    N = len(poses)
    feature_vectors = []
    for R, t in poses:
        R_T = [[R[0][0], R[1][0], R[2][0]],
               [R[0][1], R[1][1], R[2][1]],
               [R[0][2], R[1][2], R[2][2]]]
        c = [-sum(R_T[j][k]*t[k] for k in range(3)) for j in range(3)]
        z = [R_T[0][2], R_T[1][2], R_T[2][2]]
        feature_vectors.append(c + z)
    
    cluster_labels = standard_kmeans(feature_vectors, K)
    
    block_images = [set() for _ in range(K)]
    
    for k in range(K):
        cluster_indices = [i for i, label in enumerate(cluster_labels) if label == k]
        if not cluster_indices:
            continue
        cluster_centers = [feature_vectors[i][:3] for i in cluster_indices]
        
        A = [min(c[j] for c in cluster_centers) for j in range(3)]
        B = [max(c[j] for c in cluster_centers) for j in range(3)]
        
        midpoint = [(A[j] + B[j]) / 2.0 for j in range(3)]
        diff = [A[j] - B[j] for j in range(3)]
        diag_len = math.sqrt(sum(d*d for d in diff))
        
        if diag_len == 0:
            A_expanded = [A[j] - s_AABB[j] * 0.001 for j in range(3)]
            B_expanded = [B[j] + s_AABB[j] * 0.001 for j in range(3)]
        else:
            direction = [d / diag_len for d in diff]
            half_len = diag_len / 2.0
            corner1 = [midpoint[j] + s_AABB[j] * direction[j] * half_len for j in range(3)]
            corner2 = [midpoint[j] - s_AABB[j] * direction[j] * half_len for j in range(3)]
            A_expanded = [min(corner1[j], corner2[j]) for j in range(3)]
            B_expanded = [max(corner1[j], corner2[j]) for j in range(3)]
            
        for i in range(N):
            center_i = feature_vectors[i][:3]
            if all(center_i[j] >= A_expanded[j] and center_i[j] <= B_expanded[j] for j in range(3)):
                block_images[k].add(i)
                
    representatives = []
    for k in range(K):
        if not block_images[k]:
            original_cluster_indices = [i for i, label in enumerate(cluster_labels) if label == k]
            best_idx = original_cluster_indices[0] if original_cluster_indices else 0
        else:
            best_idx = None
            max_var = -1.0
            for i in block_images[k]:
                laplacian_img = compute_laplacian(images[i])
                mean_val = sum(laplacian_img) / len(laplacian_img)
                var = sum((p - mean_val)**2 for p in laplacian_img) / len(laplacian_img)
                if var > max_var:
                    max_var = var
                    best_idx = i
        representatives.append(images[best_idx])
    return representatives

def compute_laplacian(image):
    h, w = len(image), len(image[0])
    laplacian = [[0.0]*w for _ in range(h)]
    for r in range(1, h - 1):
        for c in range(1, w - 1):
            laplacian[r][c] = (image[r-1][c] + image[r+1][c] +
                               image[r][c-1] + image[r][c+1] -
                               4.0 * image[r][c])
    return laplacian

def standard_kmeans(X, K, max_iter=300, tol=1e-4):
    N, D = len(X), len(X[0])
    indices = [i for i in range(N)]
    import random
    random.shuffle(indices)
    centroids = [X[i][:] for i in indices[:K]]
    labels = [0]*N
    for _ in range(max_iter):
        new_labels = []
        for x in X:
            dists = [math.sqrt(sum((x[j]-centroids[k][j])**2 for j in range(D))) for k in range(K)]
            new_labels.append(dists.index(min(dists)))
        if new_labels == labels:
            break
        labels = new_labels
        new_centroids = [[0.0]*D for _ in range(K)]
        counts = [0]*K
        for i, label in enumerate(labels):
            counts[label] += 1
            for j in range(D):
                new_centroids[label][j] += X[i][j]
        for k in range(K):
            if counts[k] > 0:
                new_centroids[k] = [new_centroids[k][j]/counts[k] for j in range(D)]
        if math.sqrt(sum((new_centroids[k][j]-centroids[k][j])**2 for k in range(K) for j in range(D))) < tol:
            break
        centroids = new_centroids
    return labels
```

## Subtopics

### [K-Means Spatial Partitioning for Camera Clustering](16768-k-means-spatial-partitioning-for-camera-clustering.md)
A data-driven spatial decomposition strategy that uses K-Means clustering on 3D camera center positions to partition large-scale scenes into sub-scenes for parallel training. It adapts to the actual distribution of viewpoints rather than using heuristic grids, providing a principled solution for decomposing large-scale scenes in 3D reconstruction tasks. Unlike [Sub-scene Extension](16800-sub-scene-extension.md), which augments cluster radii to ensure overlap, this method performs only static spatial partitioning based on camera center positions.

## Operating characteristics
- **Time Complexity:** Not given in the sources. [G]
- **Memory Complexity:** Not given in the sources. [G]
- **Strengths:** Robust in texture-less regions compared to SfM-based splitting [S2, unofficial]. Ensures geometric coverage by clustering positions and directions [S1]. Computationally efficient [S1].
- **Limitations:** Requires known camera poses (not suitable for pre-SfM stages) [S2, unofficial]. Blurring can occur if blocks are too close and blending methods like Inverse Distance Weighting (IDW) are used [S2, unofficial].
- **When to use:** When training a second round of reconstruction/denoising with a fixed number of input views (e.g., 12) and needing to ensure diversity and sharpness [S1].
- **When not to use:** When camera poses are unknown or unreliable [S2, unofficial].

## Background
Originating from methods for large-scale NeRF reconstruction and sparse-view Gaussian Splatting, this technique addresses the scalability and stability issues of training on all available views [S1][S2, unofficial]. [S1] introduces it for "Denoise-GS" to select representative viewpoints for fine-grained denoising. [S2, unofficial] introduces "Block Splitting" for "SCALAR-NeRF" to enable distributed training of local NeRF models. It differs from Mega-NeRF by using pose priors instead of feature-metric correspondences [S2, unofficial].

## Sources
[S1] Denoise-GS: Self-Supervised Denoising for Sparse-View 3D Gaussian Splatting., https://doi.org/10.3390/s26020651, fetched 2026-10-26
[S2] SCALAR-NeRF: SCAlable LARge-scale Neural Radiance Fields for Scene Reconstruction, https://arxiv.org/abs/2311.16657v1, fetched 2026-10-26 (unofficial)
