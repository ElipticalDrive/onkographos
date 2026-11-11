# Voxels to Points Projection

Also called: voxel lifting, voxel-to-point feature projection

## Overview
Projects features from a dense voxel grid back to specific points in a point cloud using pre-computed segment IDs or voxel mappings [S1, S2, unofficial]. It solves the problem of transferring dense spatial representations (voxels) to sparse geometric representations (points) for downstream tasks like classification or segmentation [S1, S2, unofficial]. Unlike [Hard Voxelization](5345-hard-voxelization.md) which aggregates points into voxels, this method reverses the flow, retrieving or interpolating voxel data for each point [S1, S2, S3, unofficial]. It differs from [Gaussian Voxelization](8884-gaussian-voxelization.md) by using discrete grid indexing rather than continuous kernel blending [S1, S2, S3, unofficial].

## How it works
**Inputs:**
1. `voxels`: A tensor of shape $[X, Y, Z, F]$ containing voxel features, where $X, Y, Z$ are grid dimensions and $F$ is the feature dimension [S1, unofficial].
2. `segment_ids` or `point_to_voxel`: A mapping tensor.
   - In the direct projection method, `segment_ids` is an integer tensor of shape $[N]$ where $N$ is the number of points, containing the linear index of the voxel cell for each point [S1, unofficial].
   - In the lifting method, `point_to_voxel` is an integer tensor of shape $[N]$ containing the voxel index for each point, where $-1$ indicates an invalid mapping [S2, unofficial].

**Outputs:**
- `point_features`: A tensor of shape $[N, F]$ containing the projected features for each point [S1, S2, unofficial].

**Data Structures:**
- The voxel grid is typically stored as a dense multi-dimensional array or flattened 1D array for indexing [S1, unofficial].
- The mapping tensor uses `int32` or `int64` indices [S1, S2, unofficial].

**Steps:**

### Method 1: Direct Segment ID Projection [S1, unofficial]
1. **Flatten the Voxel Grid**: Reshape the voxel tensor from $[X, Y, Z, F]$ to $[X \cdot Y \cdot Z, F]$. This creates a list of all voxel feature vectors [S1, unofficial].
   $$ \text{flattened\_voxels} = \text{reshape}(\text{voxels}, [-1, F]) $$
2. **Gather Features**: Use the `segment_ids` tensor to index into the flattened voxel array. For each point $i$, retrieve the feature vector at index `segment_ids[i]` [S1, unofficial].
   $$ \text{point\_features}[i] = \text{flattened\_voxels}[\text{segment\_ids}[i]] $$

### Method 2: Voxel Value Lifting with Invalid Handling [S2, unofficial]
1. **Retrieve Mapping**: Obtain the `point_to_voxel` mapping array where each entry corresponds to a point's voxel index [S2, unofficial].
2. **Handle Invalid Indices**: Identify points where `point_to_voxel[i] == -1`. These points have invalid coordinates or were not assigned to a voxel [S2, unofficial].
3. **Index Voxel Values**: For valid indices ($>-1$), gather the corresponding voxel values/features [S2, unofficial].
4. **Assign Fill Values**: For points with index $-1$, assign a default fill value (e.g., an ignore label or zero vector) to prevent out-of-bounds errors [S2, unofficial].

### Method 3: KNN Interpolation [S3, unofficial]
1. **Batch Splitting**: Split voxel coordinates and features by batch ID into lists `voxel_coords_split` and `voxel_feats_split` [S3, unofficial].
2. **Padding**: Compute the length of each batch's voxel list and pad them to a maximum length $T_{max}$ using zero-padding [S3, unofficial].
3. **KNN Search**: Perform K-Nearest Neighbors search between the original points and the padded voxel coordinates [S3, unofficial].
   $$ \text{idx} = \text{knn\_points}(\text{original\_pts}, \text{voxel\_coords\_padded}, K=K_{interp}) $$
4. **Gather Features**: Retrieve features from the $K$ nearest voxels [S3, unofficial].
   $$ \text{gathered\_feats} = \text{knn\_gather}(\text{voxel\_feats\_padded}, \text{idx}) $$
5. **Compute Weights**: Calculate interpolation weights as the reciprocal of distances, clamped to a minimum of $\epsilon = 10^{-8}$ to prevent division by zero [S3, unofficial].
   $$ w_i = \frac{1}{\max(d_i, \epsilon)} $$
6. **Normalize Weights**: Normalize weights by summing over the $K$ dimension [S3, unofficial].
   $$ \hat{w}_i = \frac{w_i}{\sum_{j=1}^{K} w_j} $$
7. **Interpolate**: Compute the final feature vector by summing the product of gathered features and normalized weights [S3, unofficial].
   $$ \text{point\_feat} = \sum_{j=1}^{K} \hat{w}_j \cdot \text{gathered\_feats}_j $$

**Parameters:**
- `voxels`: Shape $[X, Y, Z, F]$, dtype `float32` [S1, unofficial].
- `segment_ids` / `point_to_voxel`: Shape $[N]$, dtype `int32` or `int64` [S1, S2, unofficial].
- `K_interp`: Number of neighbors for interpolation, default varies by implementation [S3, unofficial].
- `chunk_size`: Number of points processed per chunk during interpolation, default 50,000 [S3, unofficial].

**Edge Cases and Pitfalls:**
- **Invalid Coordinates**: Points with coordinates outside the voxel grid bounds or NaN/Inf values must be handled. In Method 2, these map to $-1$ and require a fill value [S2, unofficial].
- **Division by Zero**: In Method 3, distances must be clamped to $\ge 10^{-8}$ to avoid NaNs in weight calculation [S3, unofficial].
- **Index Out of Bounds**: Ensure `segment_ids` and `point_to_voxel` do not exceed the total number of voxels ($X \cdot Y \cdot Z$) [S1, S2, unofficial].

## Subtopics

### Voxel Value Lifting
Maps per-voxel values back to original point rows using a point-to-voxel mapping array [S2, unofficial]. It explicitly handles invalid mappings (index -1) by assigning a fill value, ensuring robustness for points outside the voxel grid [S2, unofficial]. Choose this when you have a pre-computed sparse voxel structure and need to lift predictions or labels back to points [S2, unofficial].

### Voxel-to-Point Interpolation via KNN
Uses K-Nearest Neighbors to interpolate features from voxel centers to point locations [S3, unofficial]. It handles batched processing and padding for variable voxel counts [S3, unofficial]. Choose this when points do not align perfectly with voxel centers and smooth interpolation is required [S3, unofficial].

## Operating characteristics
- **Time Complexity**:
  - Direct Projection: $O(N)$ for gathering, where $N$ is the number of points [S1, unofficial].
  - KNN Interpolation: $O(N \cdot K \cdot \log M)$ or $O(N \cdot K)$ depending on the KNN implementation, where $M$ is the number of voxels [S3, unofficial].
- **Memory Complexity**:
  - Direct Projection: $O(X \cdot Y \cdot Z \cdot F)$ for the voxel grid plus $O(N)$ for the output [S1, unofficial].
  - KNN Interpolation: Additional memory for padding and storing KNN indices [S3, unofficial].
- **Performance**: Direct projection is extremely fast and memory-efficient for aligned grids [S1, unofficial]. KNN interpolation is slower due to distance calculations but more robust to misalignment [S3, unofficial].
- **Strengths**: Simple implementation, low latency for direct projection [S1, unofficial]. Robustness to grid misalignment with KNN [S3, unofficial].
- **Limitations**: Direct projection fails if points are not aligned with voxel centers [S1, unofficial]. KNN interpolation is computationally expensive for large $K$ or $N$ [S3, unofficial].
- **When to Use**: Use direct projection for regular grids with aligned points [S1, unofficial]. Use KNN for irregular point distributions or when smooth transitions are needed [S3, unofficial].

## Background
Originated in 3D deep learning frameworks for processing point clouds and voxel grids [S1, S2, S3, unofficial]. The direct projection method is commonly used in TensorFlow-based 3D detection pipelines [S1, unofficial]. Voxel lifting is used in point cloud core libraries for handling sparse voxel representations [S2, unofficial]. KNN-based interpolation is used in vision-geometry fusion tasks [S3, unofficial].

## Sources
[S1] https://github.com/google-research/google-research/blob/master/tf3d/utils/voxel_utils.py#L305, https://github.com/google-research/google-research/blob/master/tf3d/utils/voxel_utils.py, fetched 2026-10-27 (unofficial)
[S2] ajavanma/pointcloud-core, https://github.com/ajavanma/pointcloud-core, fetched 2026-10-27 (unofficial)
[S3] Zhaoyibinn/VGVGGT, https://github.com/Zhaoyibinn/VGVGGT, fetched 2026-10-27 (unofficial)
