# Pose Grouping and Representative View Selection

Also called: AABB Expansion, Block Splitting, Clustering-based View Selection, KMeans Block Splitting, Representative View Selection

## Overview
Pose Grouping and Representative View Selection is a method for selecting a subset of representative training viewpoints from a larger set of rendered images by clustering camera poses and filtering for image sharpness [S1]. It addresses the problem of visual inconsistencies, amplified exposure differences, and noise propagation that arise when using all available viewpoints for subsequent training rounds [S1]. Unlike nearest neighbor selection, it groups cameras into overlapping spatial blocks using KMeans clustering on camera centers and axis-aligned bounding box (AABB) expansion, ensuring better geometric coverage and robustness in texture-less regions compared to SfM-based splitting methods [S2, S2, unofficial].

## How it works
**Inputs:**
- Set of $N$ camera poses $P = \{P_1, \dots, P_N\}$, where each pose $P_i$ consists of a rotation matrix $R_i \in \mathbb{R}^{3 \times 3}$ and translation vector $t_i \in \mathbb{R}^3$.
- Set of rendered images $I = \{I_1, \dots, I_N\}$ corresponding to the poses.
- Number of target clusters/blocks $K$ (e.g., $K=12$) [S1].
- AABB scale factor vector $s_{AABB} = [s_x, s_y, s_z]^T$ [S2, unofficial]. Default is $s_x=s_y=s_z$ [S2, unofficial].

**Outputs:**
- A set of $K$ intersected blocks $B = \{B_1, \dots, B_K\}$, where each block $B_k$ contains a subset of image indices.
- A set of $K$ representative images, one selected from each block.

**Data Structures:**
- Camera Center $c_i \in \mathbb{R}^3$.
- Forward Direction $z_i \in \mathbb{R}^3$.
- 6D Feature Vector $f_i = [c_i, z_i] \in \mathbb{R}^6$.
- Axis-Aligned Bounding Box $AABB_k = [A_k, B_k]$, where $A_k, B_k \in \mathbb{R}^3$ are the minimum and maximum corners.

**Steps:**

1. **Extract Geometric Features:**
   For each camera pose $P_i$ with extrinsic parameters $R_i, t_i$:
   - Compute camera center: $c_i = -R_i^T t_i$ [S1].
   - Compute forward direction: $z_i = R_i^T [0, 0, 1]^T$ [S1].
   - Form the 6D feature vector: $f_i = [c_i, z_i]$ [S1].

2. **Initial Clustering:**
   Perform KMeans clustering on the set of camera centers $\{t_1, \dots, t_N\}$ (or equivalently $\{c_1, \dots, c_N\}$ as they are rigidly related) to assign an initial block ID $L_i$ to each image [S2, unofficial].
   $$ L_i = \text{KMeans}(\{t_1, \dots, t_N\}, K) $$
   This creates $K$ initial non-overlapping blocks based on spatial proximity of camera centers [S2, unofficial].

3. **Compute Bounding Boxes:**
   For each block $k \in [1, K]$:
   - Let $P_k$ be the set of camera poses in block $k$.
   - Compute the axis-aligned bounding box $AABB_k = [A_k, B_k]$ where:
     $$ A_k = \min(P_k), \quad B_k = \max(P_k) $$
     Here, min/max are computed component-wise over the 3D coordinates of the camera poses in the block [S2, unofficial].

4. **Expand Bounding Boxes:**
   Compute the diagonal midpoint $C_k$ and direction $r_k$ for $AABB_k$:
   $$ C_k = \frac{A_k + B_k}{2} $$
   $$ r_k = \frac{B_k - A_k}{\|B_k - A_k\|} $$
   $$ \text{len}_k = \frac{\|B_k - A_k\|}{2} $$
   Recompute the expanded bounds $A'_k$ and $B'_k$ using the scale factor $s_{AABB}$:
   $$ A'_k = C_k + s_{AABB} \cdot r_k \cdot \text{len}_k $$
   $$ B'_k = C_k - s_{AABB} \cdot r_k \cdot \text{len}_k $$
   Note: The source formula (8) in [S2, unofficial] defines $A = C + s_{AABB} \cdot r \cdot \text{len}$ and $B = C - s_{AABB} \cdot r \cdot \text{len}$. Depending on the definition of $r$ (from A to B or B to A), the signs may swap, but the intent is to expand the box along the diagonal axis by the factor $s_{AABB}$ [S2, unofficial]. If $s_{AABB} = 1.0$, there is no overlap; if $s_{AABB} > 1.0$, blocks overlap [S2, unofficial].

5. **Regroup Images:**
   Initialize empty blocks $B'_k$. For each image $i$:
   - Check if its camera pose $P_i$ lies within the expanded bounds $[A'_k, B'_k]$ for any block $k$.
   - If $A'_k \le P_i \le B'_k$ (component-wise), add image $i$ to block $B'_k$.
   - An image may belong to multiple blocks due to overlap [S2, unofficial].

6. **Select Representative Views:**
   For each final block $B'_k$:
   - Calculate the Laplacian variance for each image $I_i$ in the block:
     $$ \text{Var}_{Lap}(I_i) = \text{Var}(\nabla^2 I_i) $$
     where $\nabla^2 I_i$ is the Laplacian gradient of the image [S1].
   - Select the image with the highest $\text{Var}_{Lap}$ as the representative view for block $k$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from sklearn.cluster import KMeans

def pose_grouping_and_representative_view_selection(poses, images, K, s_AABB):
    """
    poses: list of tuples (R, t) of length N
    images: list of image arrays of length N
    K: int, number of clusters
    s_AABB: float or array-like, scale factor for AABB expansion
    """
    N = len(poses)
    
    # Step 1: Extract features
    centers = []
    directions = []
    for R, t in poses:
        c = -R.T @ t
        z = R.T @ np.array([0, 0, 1])
        centers.append(c)
        directions.append(z)
    
    centers = np.array(centers)
    
    # Step 2: Initial Clustering on centers
    kmeans = KMeans(n_clusters=K, random_state=42)
    labels = kmeans.fit_predict(centers)
    
    # Step 3 & 4: Compute and Expand AABBs
    expanded_bounds = []
    for k in range(K):
        # Get poses in this cluster
        cluster_indices = np.where(labels == k)[0]
        if len(cluster_indices) == 0:
            expanded_bounds.append((None, None))
            continue
            
        cluster_poses = centers[cluster_indices]
        
        # Compute AABB
        A = np.min(cluster_poses, axis=0)
        B = np.max(cluster_poses, axis=0)
        
        # Compute midpoint, direction, half-length
        C = (A + B) / 2.0
        diag_vec = B - A
        length = np.linalg.norm(diag_vec)
        
        if length == 0:
            # Degenerate case: all points are the same
            expanded_bounds.append((A, B))
            continue
            
        r = diag_vec / length
        half_len = length / 2.0
        
        # Expand
        # Note: Source [S2, unofficial] uses s_AABB * r * len. 
        # Assuming s_AABB is a scalar or broadcastable vector.
        # If s_AABB is a vector [sx, sy, sz], the expansion is anisotropic.
        # The formula A = C + s * r * len implies expansion along the diagonal.
        # However, standard AABB expansion is usually axis-aligned.
        # The source explicitly says "Compute the ray direction of the diagonal... 
        # Recompute A and B by...". This implies the box is rotated to align with the diagonal?
        # No, AABB is Axis-Aligned. The formula in [S2, unofficial] Eq 8 is:
        # A = C + s_AABB * r * len
        # B = C - s_AABB * r * len
        # This actually rotates the box if r is not axis-aligned. 
        # But the text says "axis-aligned bounding boxes". 
        # Let's stick to the literal formula provided in [S2, unofficial] for the "Regroup" step 
        # which checks if Pi is within [A, B].
        
        # If s_AABB is a scalar:
        if np.isscalar(s_AABB):
            offset = s_AABB * r * half_len * 2 # len in formula is half-length? 
            # Source: len = ||B-A||/2. Formula: A = C + s * r * len.
            # So offset is s * r * (||B-A||/2).
            # Wait, Eq 8: A = C + s * r * len. B = C - s * r * len.
            # So the new box is centered at C, with half-diagonal length s * len.
            # This creates a box aligned with the diagonal, not the axes.
            # However, the regrouping step says "If image i's pose Pi is within the new bounds [A, B]".
            # Usually "within bounds" for a rotated box is complex.
            # Let's assume the source implies an axis-aligned expansion based on the component-wise min/max 
            # or that the "bounds" check is done against the rotated box.
            # Given the ambiguity, we implement the literal formula for A and B.
            
            new_A = C + s_AABB * r * half_len
            new_B = C - s_AABB * r * half_len
        else:
            # If s_AABB is a vector, the operation is element-wise?
            # The source says s_AABB = [sx, sy, sz].
            # And r is a vector.
            # The multiplication s_AABB * r * len is likely element-wise if dimensions match.
            # But r is a direction vector.
            # Let's assume scalar for simplicity as default is sx=sy=sz.
            new_A = C + s_AABB * r * half_len
            new_B = C - s_AABB * r * half_len
            
        expanded_bounds.append((new_A, new_B))

    # Step 5: Regroup
    blocks = [[] for _ in range(K)]
    for i in range(N):
        pose_i = centers[i]
        for k in range(K):
            A_k, B_k = expanded_bounds[k]
            if A_k is None:
                continue
            # Check if pose_i is within [A_k, B_k]
            # Assuming component-wise comparison for AABB-like check, 
            # or dot product for rotated box. 
            # Source says "within the new bounds [A, B]". 
            # Given it's called AABB, let's assume component-wise min/max check 
            # is the intended "simple" implementation, despite the diagonal rotation formula.
            # However, Eq 8 explicitly rotates. 
            # Let's implement the check as: min(A_k, B_k) <= pose_i <= max(A_k, B_k) component-wise.
            # This is the standard interpretation of "within bounds" for a box defined by corners.
            lower = np.minimum(A_k, B_k)
            upper = np.maximum(A_k, B_k)
            if np.all(lower <= pose_i) and np.all(pose_i <= upper):
                blocks[k].append(i)

    # Step 6: Select Representatives
    representatives = []
    for k in range(K):
        if not blocks[k]:
            representatives.append(None)
            continue
        
        best_idx = -1
        best_var = -np.inf
        
        for i in blocks[k]:
            # Compute Laplacian Variance
            # Laplacian kernel (3x3)
            laplacian_kernel = np.array([[0, 1, 0], 
                                         [1, -4, 1], 
                                         [0, 1, 0]])
            # Apply convolution (simplified)
            # In practice, use scipy.signal.convolve2d or similar
            # Here we assume a function exists
            lap_img = apply_laplacian(images[i])
            var_lap = np.var(lap_img)
            
            if var_lap > best_var:
                best_var = var_lap
                best_idx = i
                
        representatives.append(best_idx)
        
    return blocks, representatives

def apply_laplacian(image):
    # Placeholder for actual convolution
    # Returns the Laplacian gradient image
    pass
```

**Parameters:**
- $K$: Number of clusters. Default $K=12$ [S1].
- $s_{AABB}$: Scale factor vector $[s_x, s_y, s_z]$. Default $s_x=s_y=s_z$. Controls overlap. $s=1.0$ means no overlap [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Degenerate Clusters:** If a cluster has only one point, the diagonal length is 0. The code must handle division by zero or zero-length diagonals [S2, unofficial].
- **Texture-less Regions:** SfM-based splitting fails here, but this method uses camera pose priors, making it robust [S2, unofficial].
- **Overlap Management:** High $s_{AABB}$ increases overlap, potentially causing redundancy. Low $s_{AABB}$ may leave gaps [S2, unofficial].

## Subtopics

### Nearest Camera Pose Selection
Selects the single most relevant training image based on minimal positional and angular distance. Simpler, but does not group views for robust coverage [12726-nearest-camera-pose-selection.md].

### Random Pose Selection
Selects views randomly. Lacks geometric structure and may miss important viewpoints [12725-random-pose-selection.md].

## Operating characteristics
- **Time Complexity:** KMeans clustering is $O(N \cdot K \cdot D \cdot I)$ where $N$ is number of images, $K$ clusters, $D$ dimensions (6), and $I$ iterations. AABB computation is $O(N)$. Regrouping is $O(N \cdot K)$. Laplacian calculation is $O(N \cdot H \cdot W)$ per block. Overall efficient for large $N$ [S1][S2, unofficial].
- **Memory Complexity:** $O(N)$ to store poses and images. $O(K)$ for blocks [S1][S2, unofficial].
- **Strengths:** Robust in texture-less regions; ensures geometric coverage via overlapping blocks; computationally efficient [S2, unofficial].
- **Limitations:** Requires pre-computed camera poses; relies on Laplacian variance which may not always correlate with geometric quality in noisy scenes [S1].
- **When to Use:** When training a second round of reconstruction/denoising and a fixed number of representative views are needed [S1].

## Background
The method was introduced in **Denoise-GS** (2024) for self-supervised denoising in sparse-view 3D Gaussian Splatting [S1]. It was also employed in **SCALAR-NeRF** (2023) for scalable large-scale NeRF reconstruction, where it was referred to as "Block Splitting" [S2, unofficial]. The approach differs from SfM-based splitting (e.g., Mega-NeRF) by leveraging known camera poses rather than feature-metric correspondences [S2, unofficial].

## Sources
[S1] Denoise-GS: Self-Supervised Denoising for Sparse-View 3D Gaussian Splatting., https://doi.org/10.3390/s26020651, fetched 2026-10-26
[S2] SCALAR-NeRF: SCAlable LARge-scale Neural Radiance Fields for Scene Reconstruction, https://arxiv.org/abs/2311.16657v1, fetched 2026-10-26 (unofficial)
