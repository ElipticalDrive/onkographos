# Mean Feature Aggregation by Voxel

Also called: Lidar Feature Extraction, Mean Feature Aggregation, Voxel Feature Averaging

## Overview
Mean Feature Aggregation by Voxel computes mean features for each voxel by aggregating point features based on voxel indices, returning unique voxel features and their indices [S2, unofficial]. It serves as a geometric feature extraction step that converts sparse, noisy LiDAR point clouds into dense, structured neural volumetric features suitable for downstream rendering tasks [S1]. Unlike its sibling [Voxel Feature Composition](7021-voxel-feature-composition.md), which generates per-voxel features by sampling Gaussian volumes and weighting by attenuation, this method relies on direct spatial averaging of raw LiDAR points followed by a 3D sparse convolutional network to encode global context [S1].

## How it works
**Inputs:**
1.  A sequence of LiDAR point clouds from multiple frames.
2.  Camera poses (extrinsics, intrinsics) and depth maps (optional, for synchronization or filtering, though the core aggregation is LiDAR-centric) [S2, unofficial].
3.  Voxel grid resolution (cell size) and bounding box dimensions.

**Outputs:**
1.  A set of unique voxel indices.
2.  A feature tensor where each occupied voxel contains an $n$-dimensional neural feature vector $f_i$ [S1].
3.  The final LiDAR embeddings $P = \{(p_i, f_i) | i = 1, \dots, N\}$, where $p_i$ is the 3D location of point $i$ and $f_i$ is the neural feature of the voxel cell it resides in [S1].

**Data Structures:**
*   **Voxel Grid:** A 3D grid defined by a bounding box and cell size. Points are mapped to integer indices $(x, y, z)$.
*   **Point Cloud:** A list of points with 3D coordinates $(x, y, z)$ and optional raw features.
*   **Voxel Features:** A tensor of shape $(N_{voxels}, n)$, where $N_{voxels}$ is the number of occupied voxels and $n$ is the feature dimension (initially 3 for spatial position, later expanded by the UNet).

**Steps:**

1.  **Aggregation:** Aggregate LiDAR point clouds from all frames of the entire sequence to construct a denser set of point clouds [S1]. This mitigates sparsity issues inherent in single-frame LiDAR scans.
2.  **Device Transfer:** Move all input tensors (points, features, batch IDs, bounding boxes, colors, depth confidence) to the target device (e.g., GPU) [S2, unofficial].
3.  **Voxelization:** Voxelize the aggregated point clouds into a voxel grid. For each voxel cell, compute the mean spatial position of all points falling into that cell. This yields an initial 3-dimensional feature vector for each voxel cell [S1].
    *   Let $V$ be the set of voxels. For a voxel $v$, let $P_v$ be the set of points $p_j$ such that $v = \text{voxel\_index}(p_j)$.
    *   The initial feature $f_v^{(0)}$ is:
        $$ f_v^{(0)} = \frac{1}{|P_v|} \sum_{p_j \in P_v} p_j $$
4.  **Index Computation:** Compute voxel indices using a function `get_vox_indices` with `shift=False` and `cat_batch_ids=True` to handle multi-batch or multi-sequence inputs correctly [S2, unofficial].
5.  **Hashing:** Compute voxel hashes using a hash function `self.hash_fn(voxel_indices)` to map 3D indices to unique integer identifiers [S2, unofficial].
6.  **Uniqueness:** Get unique hashes and inverse IDs using `torch.unique(voxel_hash, return_inverse=True)` to identify distinct occupied voxels and map points back to their unique voxel IDs [S2, unofficial].
7.  **Feature Encoding:** Encode the scene geometry feature using a 3D sparse UNet on the voxel grid [S1].
    *   **Input:** The voxel grid along with its initial 3-dim features (mean positions).
    *   **Process:** The 3D sparse UNet [10, 33] processes the sparse tensor, learning from the global context of the scene geometry [S1].
    *   **Output:** Neural volumetric features consisting of an $n$-dimensional feature for each occupied voxel [S1].
8.  **Embedding Construction:** Associate each original LiDAR point $p_i$ with the neural feature $f_i$ of the voxel cell it resides in, forming the final LiDAR embeddings $P$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def mean_feature_aggregation_by_voxel(points_all_frames, voxel_size, bounding_box, sparse_unet_model):
    """
    Aggregates LiDAR points into voxels and encodes features using a 3D sparse UNet.
    
    Args:
        points_all_frames: List of tensors, each of shape (N, 3) containing 3D coordinates.
        voxel_size: Float, size of each voxel cell.
        bounding_box: Tuple of (min_x, min_y, min_z, max_x, max_y, max_z).
        sparse_unet_model: A 3D sparse UNet model instance.
        
    Returns:
        voxel_features: Tensor of shape (N_voxels, n) containing neural features.
        voxel_indices: Tensor of shape (N_voxels, 3) containing voxel coordinates.
        embeddings: List of tuples (point_coord, voxel_feature) for each original point.
    """
    # Step 1: Aggregate points from all frames
    all_points = torch.cat(points_all_frames, dim=0) # Shape: (N_total, 3)
    
    # Step 2: Move to device (assumed device is 'cuda')
    device = all_points.device
    all_points = all_points.to(device)
    
    # Step 3: Voxelize
    # Compute voxel indices for each point
    # voxel_index = floor((point - min_bound) / voxel_size)
    min_bound = torch.tensor([bounding_box[0], bounding_box[1], bounding_box[2]], device=device)
    voxel_indices = ((all_points - min_bound) / voxel_size).long()
    
    # Filter points outside bounding box
    max_bound = torch.tensor([bounding_box[3], bounding_box[4], bounding_box[5]], device=device)
    max_indices = ((max_bound - min_bound) / voxel_size).long()
    valid_mask = (voxel_indices >= 0).all(dim=1) & (voxel_indices <= max_indices).all(dim=1)
    valid_points = all_points[valid_mask]
    valid_voxel_indices = voxel_indices[valid_mask]
    
    # Compute mean spatial position for each voxel
    # Group points by voxel index
    unique_voxels, inverse_indices = torch.unique(valid_voxel_indices, return_inverse=True, sorted=True)
    
    # Initialize mean features with zeros
    num_voxels = unique_voxels.shape[0]
    mean_features = torch.zeros((num_voxels, 3), device=device)
    counts = torch.zeros((num_voxels,), device=device)
    
    # Scatter add for sum and count
    # Note: In practice, use scatter_add_ for efficiency
    mean_features.scatter_add_(0, inverse_indices.unsqueeze(1).expand(-1, 3), valid_points)
    counts.scatter_add_(0, inverse_indices, torch.ones_like(inverse_indices, dtype=torch.float))
    
    # Compute mean
    mean_features = mean_features / counts.unsqueeze(1)
    
    # Step 4 & 5: Compute hashes (simplified for pseudocode)
    # In real implementation, use self.hash_fn(voxel_indices)
    # Here we assume unique_voxels are already the unique identifiers
    voxel_hashes = unique_voxels # Placeholder for hash function output
    
    # Step 6: Get unique hashes and inverse IDs
    # Already done via torch.unique above
    
    # Step 7: Encode with 3D Sparse UNet
    # Create sparse tensor from mean features
    # SparseTensor requires coordinates and features
    # coordinates = unique_voxels (batch index 0 assumed)
    sparse_tensor = SparseTensor(features=mean_features, coordinates=unique_voxels.unsqueeze(0)) # Batch dim added
    
    # Forward pass through UNet
    encoded_features = sparse_unet_model(sparse_tensor)
    
    # Extract features from the output sparse tensor
    voxel_features = encoded_tensor.features # Shape: (N_voxels, n)
    
    # Step 8: Construct embeddings
    # Map each original point to its voxel's encoded feature
    # Use inverse_indices to look up the encoded feature for each valid point
    point_embeddings = []
    for i, point in enumerate(valid_points):
        voxel_idx = inverse_indices[i]
        feature = voxel_features[voxel_idx]
        point_embeddings.append((point, feature))
        
    return voxel_features, unique_voxels, point_embeddings
```

**Parameters:**
*   `voxel_size`: Not given in the sources. Typical values in 3D perception range from 0.05m to 0.1m for street scenes [S1].
*   `n` (feature dimension): Not given in the sources. Determined by the output channels of the 3D sparse UNet.

**Edge Cases and Pitfalls:**
*   **Empty Voxels:** Voxels with no points are excluded from the unique set. The sparse UNet handles sparsity efficiently, but empty regions have no features [S1].
*   **Precision:** Floating-point precision in voxel index calculation can lead to boundary errors. Using `long()` truncation is standard but may cause points near boundaries to be assigned to incorrect voxels [G].
*   **Memory:** The 3D sparse UNet requires significant memory for large scenes. The sparsity of the voxel grid is crucial for efficiency [S1].

## Operating characteristics
*   **Time Complexity:** Dominated by the 3D sparse UNet forward pass. Voxelization is $O(N)$ where $N$ is the number of points. Hashing and unique operations are $O(N \log N)$ or $O(N)$ with hash tables.
*   **Memory Complexity:** Proportional to the number of occupied voxels and the feature dimension $n$. Sparse data structures minimize memory usage for empty regions.
*   **Strengths:** Produces dense, globally consistent geometric features from sparse LiDAR data. The 3D sparse UNet captures local and global context [S1].
*   **Limitations:** Relies on the quality and density of the input LiDAR data. Aggregation across frames assumes static scenes or requires precise synchronization [S1].
*   **When to use:** When LiDAR data is available and geometric context is critical for rendering, especially in large-scale outdoor scenes like street environments [S1].
*   **When not to use:** When only RGB images are available, or when real-time performance is critical and the overhead of the sparse UNet is prohibitive.

## Background
Originating from the LidaRF paper [S1], this method was introduced to leverage the strength of LiDAR data for neural radiance fields in street scenes. It builds on the success of 3D sparse UNets in 3D perception frameworks [10, 33, 22, 47] to encode scene geometry. The approach was developed to address the sparsity and noise of single-frame LiDAR scans by aggregating data across sequences and encoding it into a dense neural representation.

## Sources
[S1] LidaRF: Delving into Lidar for Neural Radiance Field on Street Scenes, https://doi.org/10.1109/cvpr52733.2024.01850, fetched 2026-10-26
[S2] Zhaoyibinn/VGVGGT, https://github.com/Zhaoyibinn/VGVGGT, fetched 2026-10-27 (unofficial)
