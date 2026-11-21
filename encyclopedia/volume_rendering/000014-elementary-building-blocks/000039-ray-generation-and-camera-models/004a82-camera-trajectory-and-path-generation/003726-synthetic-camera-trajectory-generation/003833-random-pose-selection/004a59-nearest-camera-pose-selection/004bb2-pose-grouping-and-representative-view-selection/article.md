# Pose Grouping and Representative View Selection

Also called: AABB Expansion, Block Splitting, Clustering-based View Selection, KMeans Block Splitting, Representative View Selection

## Overview
This method selects a subset of representative training viewpoints from a large set of rendered images by clustering camera poses and filtering for image quality. It addresses the problem of visual inconsistencies, amplified exposure differences, and noise propagation that arise when using all available frames for subsequent training rounds [S1]. Unlike its parent [Nearest Camera Pose Selection](12726-nearest-camera-pose-selection.md), which selects the single most relevant image based on minimal distance, this method partitions the entire dataset into groups to ensure geometric coverage and selects the sharpest image from each group [S1]. It differs from SfM-based splitting methods (e.g., Mega-NeRF) by leveraging known camera pose priors and KMeans clustering on centers rather than feature-metric correspondences, avoiding failures in texture-less regions [S2, unofficial].

## How it works
**Inputs:**
- Set of $N$ camera poses $P = \{P_1, \dots, P_N\}$, where each $P_i$ consists of rotation matrix $R_i$ and translation vector $t_i$.
- Set of rendered images $I = \{I_1, \dots, I_N\}$ corresponding to the poses.
- Number of clusters $K$ (typically 12) [S1].
- Bounding box scale factor vector $s_{AABB} = [s_x, s_y, s_z]^T$ (default $s_x=s_y=s_z=1.0$) [S2, unofficial].

**Outputs:**
- A set of $K$ representative images, one from each cluster/block.

**Data Structures:**
- **Camera Center:** $c_n = -R_n^T t_n$ [S1].
- **Forward Direction:** $z_n = R_n^T [0,0,1]^T$ [S1].
- **Feature Vector:** $v_n = [c_n, z_n] \in \mathbb{R}^6$ [S1].
- **Axis-Aligned Bounding Box (AABB):** Defined by minimum corner $A$ and maximum corner $B$.

**Steps:**

1. **Extract Geometric Features:**
   For each image $i$, compute the camera center $c_i$ and forward direction $z_i$ from the extrinsic parameters $R_i, t_i$. Construct the 6D feature vector $v_i = [c_i, z_i]$ [S1].

2. **Initial Clustering:**
   Perform KMeans clustering on the set of feature vectors $\{v_i\}$ with $K$ clusters. Assign each image $i$ to an initial block $B_k$ based on its cluster ID [S1][S2, unofficial].
   $$ LI_i = \text{KMeans}(v_i, K) $$
   where $LI_i$ is the block ID of image $i$ [S2, unofficial].

3. **Compute Bounding Boxes:**
   For each cluster $k$, determine the spatial extent of the camera centers $P_k = \{P_i \mid LI_i = k\}$. Compute the initial AABB bounds $A_k$ and $B_k$:
   $$ A_k = \min(P_k), \quad B_k = \max(P_k) $$
   where min/max are taken component-wise over the 3D coordinates of the camera centers in the cluster [S2, unofficial].

4. **Expand Bounding Boxes (AABB Expansion):**
   To introduce overlap between blocks, expand the AABBs using the scale factor $s_{AABB}$.
   - Compute the diagonal midpoint $C_k$:
     $$ C_k = \frac{A_k + B_k}{2} $$
   - Compute the diagonal direction $r_k$ and half-length $len_k$:
     $$ r_k = \frac{B_k - A_k}{\|B_k - A_k\|}, \quad len_k = \frac{\|B_k - A_k\|}{2} $$
   - Recompute the expanded bounds $A'_k$ and $B'_k$:
     $$ A'_k = C_k + s_{AABB} \cdot r_k \cdot len_k $$
     $$ B'_k = C_k - s_{AABB} \cdot r_k \cdot len_k $$
   Note: The source notation uses $A$ and $B$ for the final bounds after expansion [S2, unofficial].

5. **Regroup Images:**
   Reassign images to blocks based on the expanded bounds. If the camera pose $P_i$ of image $i$ lies within the expanded bounds $[A'_k, B'_k]$ (component-wise), add image $i$ to block $B_k$. This allows images to belong to multiple overlapping blocks [S2, unofficial].
   $$ \text{If } A'_k \le P_i \le B'_k, \text{ then } B_k \leftarrow B_k \cup \{I_i\} $$

6. **Select Representative Views:**
   For each block $B_k$, calculate the Laplacian variance for every image $I_i \in B_k$. Select the image with the highest variance as the representative view for that block.
   $$ \text{Representative}_k = \arg \max_{I_i \in B_k} \text{Var}_{Lap}(I_i) $$

**Formula:**
The sharpness metric is the variance of the Laplacian gradient:
$$ \text{Var}_{Lap}(I) = \text{Var}(\nabla^2 I) $$
where $\nabla^2 I$ is the Laplacian gradient of image $I$. Higher values indicate richer high-frequency details (texture/edges) [S1].

**Parameters:**
- $K$: Number of clusters/blocks. Default 12 [S1].
- $s_{AABB}$: Scale factor vector. Default $[1, 1, 1]^T$ (no overlap) [S2, unofficial].
- $\gamma$: Hyperparameter for Inverse Distance Weighting (used in blending, not selection). Default 1 [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Texture-less Regions:** SfM-based splitting fails here; this method uses pose priors, making it robust [S2, unofficial].
- **Overlap Ratio:** If $s_{AABB} \to \infty$, blocks encompass all training images. If $s_{AABB} = 1.0$, there is no overlap [S2, unofficial].
- **Blurring:** If blocks are too close, blending methods like Inverse Distance Weighting (IDW) can produce blurry images due to similar weights for outliers [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from sklearn.cluster import KMeans

def pose_grouping_and_representative_view_selection(poses, images, K=12, s_AABB=np.array([1.0, 1.0, 1.0])):
    """
    Selects representative views by clustering camera poses and expanding bounding boxes.
    
    Args:
        poses: List of tuples (R, t) for N images. R is 3x3 rotation, t is 3x1 translation.
        images: List of image arrays (for Laplacian calculation).
        K: Number of clusters.
        s_AABB: Scale factor vector for AABB expansion.
        
    Returns:
        List of K representative images.
    """
    N = len(poses)
    
    # Step 1: Extract geometric features
    # c_n = -R_n^T * t_n
    # z_n = R_n^T * [0, 0, 1]^T
    feature_vectors = []
    for R, t in poses:
        R_T = R.T
        c = -R_T @ t
        z = R_T @ np.array([0, 0, 1])
        # Concatenate to form 6D vector
        v = np.concatenate([c, z])
        feature_vectors.append(v)
    
    feature_matrix = np.array(feature_vectors) # Shape (N, 6)
    
    # Step 2: Initial Clustering
    kmeans = KMeans(n_clusters=K, random_state=42)
    cluster_labels = kmeans.fit_predict(feature_matrix)
    
    # Step 3 & 4: Compute and Expand Bounding Boxes
    # We need to track which images fall into which expanded AABBs
    # Initialize block membership as empty sets
    block_images = [set() for _ in range(K)]
    
    for k in range(K):
        # Get indices of images in cluster k
        cluster_indices = np.where(cluster_labels == k)[0]
        if len(cluster_indices) == 0:
            continue
            
        # Get camera centers for this cluster
        # Camera center is the first 3 components of the feature vector
        cluster_centers = feature_matrix[cluster_indices, :3]
        
        # Compute initial AABB bounds (min and max of centers)
        A = np.min(cluster_centers, axis=0)
        B = np.max(cluster_centers, axis=0)
        
        # Compute diagonal properties
        midpoint = (A + B) / 2.0
        diagonal_vec = B - A
        diagonal_len = np.linalg.norm(diagonal_vec)
        
        if diagonal_len == 0:
            # Degenerate case: all centers are identical
            # Use a small epsilon or just the point itself
            A_expanded = A - s_AABB * 0.001
            B_expanded = B + s_AABB * 0.001
        else:
            direction = diagonal_vec / diagonal_len
            half_len = diagonal_len / 2.0
            
            # Expand bounds
            # A' = C + s * r * len
            # B' = C - s * r * len
            # Note: Source says A = C + s * r * len and B = C - s * r * len
            # But typically A is min and B is max. 
            # If s > 1, the expansion might flip min/max depending on direction.
            # We compute the new bounds and then take min/max to ensure A <= B component-wise
            # for the containment check.
            
            # Actually, the source defines A and B as the corners.
            # Let's compute the two corners explicitly.
            corner1 = midpoint + s_AABB * direction * half_len
            corner2 = midpoint - s_AABB * direction * half_len
            
            A_expanded = np.minimum(corner1, corner2)
            B_expanded = np.maximum(corner1, corner2)
            
        # Step 5: Regroup images based on expanded AABB
        for i in range(N):
            # Check if camera center of image i is within [A_expanded, B_expanded]
            center_i = feature_matrix[i, :3]
            if np.all(center_i >= A_expanded) and np.all(center_i <= B_expanded):
                block_images[k].add(i)
                
    # Step 6: Select representative views
    representatives = []
    for k in range(K):
        if not block_images[k]:
            # Fallback: if a block is empty, pick a random image or the cluster centroid's image
            # This is a gap not explicitly covered, so we pick the first image of the original cluster
            original_cluster_indices = np.where(cluster_labels == k)[0]
            if len(original_cluster_indices) > 0:
                idx = original_cluster_indices[0]
            else:
                idx = 0 # Fallback
        else:
            # Calculate Laplacian variance for each image in the block
            best_idx = None
            max_var = -1
            
            for i in block_images[k]:
                # Compute Laplacian variance
                # Laplacian gradient is approximated by standard kernel
                # Var(Laplacian(I))
                laplacian_img = compute_laplacian(images[i])
                var = np.var(laplacian_img)
                
                if var > max_var:
                    max_var = var
                    best_idx = i
                    
        representatives.append(images[best_idx])
        
    return representatives

def compute_laplacian(image):
    """
    Computes the Laplacian gradient of an image.
    Uses a standard 3x3 Laplacian kernel.
    """
    # Convert to grayscale if necessary
    if len(image.shape) == 3:
        gray = np.mean(image, axis=2)
    else:
        gray = image
        
    # Normalize to 0-1
    gray = gray.astype(np.float64)
    if gray.max() > 1.0:
        gray = gray / 255.0
        
    # Apply Laplacian kernel
    kernel = np.array([[0, 1, 0],
                       [1, -4, 1],
                       [0, 1, 0]])
                       
    # Convolve
    from scipy.signal import convolve2d
    laplacian = convolve2d(gray, kernel, mode='same', boundary='symm')
    
    return laplacian
```

## Subtopics

### [K-Means Spatial Partitioning for Camera Clustering](16768-k-means-spatial-partitioning-for-camera-clustering.md)
A data-driven spatial decomposition strategy that uses K-Means clustering on 3D camera center positions to partition large-scale scenes into sub-scenes for parallel training. It adapts to the actual distribution of viewpoints rather than using heuristic grids, providing a principled solution for decomposing large-scale scenes in 3D reconstruction tasks. Unlike [Sub-scene Extension](16800-sub-scene-extension.md), which augments cluster radii to ensure overlap, this method performs only static spatial partitioning based on camera center positions.

## Operating characteristics
- **Time Complexity:** Dominated by KMeans clustering, which is $O(N \cdot K \cdot D \cdot I)$ where $N$ is number of images, $K$ is clusters, $D$ is dimension (6), and $I$ is iterations. Regrouping is $O(N \cdot K)$ for checking bounds. Laplacian calculation is $O(N \cdot H \cdot W)$ per image.
- **Memory Complexity:** $O(N \cdot D)$ for feature vectors. $O(N)$ for block assignments.
- **Strengths:** Robust in texture-less regions compared to SfM-based splitting [S2, unofficial]. Ensures geometric coverage by clustering positions and directions [S1]. Computationally efficient [S1].
- **Limitations:** Requires known camera poses (not suitable for pre-SfM stages) [S2, unofficial]. Blurring can occur if blocks are too close and blending is used [S2, unofficial].
- **When to use:** When training a second round of reconstruction/denoising with a fixed number of input views (e.g., 12) and needing to ensure diversity and sharpness [S1].
- **When not to use:** When camera poses are unknown or unreliable [S2, unofficial].

## Background
Originating from methods for large-scale NeRF reconstruction and sparse-view Gaussian Splatting, this technique addresses the scalability and stability issues of training on all available views. [S1] introduces it for "Denoise-GS" to select representative viewpoints for fine-grained denoising. [S2, unofficial] introduces "Block Splitting" for "SCALAR-NeRF" to enable distributed training of local NeRF models. It differs from Mega-NeRF by using pose priors instead of feature-metric correspondences [S2, unofficial].

## Sources
[S1] Denoise-GS: Self-Supervised Denoising for Sparse-View 3D Gaussian Splatting., https://doi.org/10.3390/s26020651, fetched 2026-10-26
[S2] SCALAR-NeRF: SCAlable LARge-scale Neural Radiance Fields for Scene Reconstruction, https://arxiv.org/abs/2311.16657v1, fetched 2026-10-26 (unofficial)
