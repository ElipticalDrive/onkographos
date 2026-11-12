# Voxel Value Lifting


## Overview
Voxel Value Lifting maps per-voxel values (such as encoded predictions or features) back to the original point rows using a pre-computed point-to-voxel mapping [S1, unofficial]. It solves the problem of reconstructing point-level data from a dense voxel grid representation, ensuring that each output row corresponds to an input point [S1, unofficial]. Unlike direct voxelization which aggregates data into grid cells, lifting distributes grid data back to specific points, preserving the original point IDs while allowing invalid points to be handled via special mapping values [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: `voxel_values`, an array of shape `[V, ...]` containing values for each voxel (e.g., encoded predictions, features).
- Input: `point_to_voxel`, an integer array of shape `[N]` mapping each of the $N$ points to a voxel index.
- Output: `lifted_values`, an array of shape `[N, ...]` where each row contains the value from the corresponding voxel.

**Data Structures:**
- `point_to_voxel`: Int64 array. Valid indices are $0 \leq i < V$. Invalid mappings use the value `-1` [S1, unofficial].
- `voxel_values`: Float64 or Int64 array depending on the feature type [S1, unofficial].

**Steps:**
1. Initialize `lifted_values` with a default fill value (e.g., ignore label or zero) for all $N$ points [S1, unofficial].
2. Iterate through each point $p$ from $0$ to $N-1$:
   a. Retrieve the voxel index $v\_idx = \text{point\_to\_voxel}[p]$.
   b. If $v\_idx \neq -1$:
      i. Assign `lifted_values[p] = voxel_values[v_idx]` [S1, unofficial].
   c. If $v\_idx == -1$:
      i. Keep the default fill value (e.g., ignore label) [S1, unofficial].

**Formulas:**
Let $P$ be the set of point indices $\{0, \dots, N-1\}$ and $V$ be the set of voxel indices $\{0, \dots, M-1\}$.
The lifting function $L: P \rightarrow \text{Values}$ is defined as:
$$
L(p) = \begin{cases} 
\text{voxel\_values}[\text{point\_to\_voxel}[p]] & \text{if } \text{point\_to\_voxel}[p] \geq 0 \\
\text{fill\_value} & \text{if } \text{point\_to\_voxel}[p] = -1 
\end{cases}
$$

**Parameters:**
- `fill_value`: Typically an ignore label (e.g., 255 in semantic segmentation) or a zero vector [S1, unofficial].

**Edge Cases and Pitfalls:**
- Points with invalid coordinates or features have a mapping of `-1` [S1, unofficial].
- Lifting must explicitly check for `-1` indices to avoid out-of-bounds access or incorrect data retrieval [S1, unofficial].
- Coordinates use floor division by voxel size; negative coordinates are handled by floor without origin offset or boundary epsilon [S1, unofficial].
- Input coordinates/features normalize to float64; mappings and point IDs use int64 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lift_voxel_values(voxel_values, point_to_voxel, fill_value=255):
    """
    Map per-voxel values back to point rows.
    
    Args:
        voxel_values: np.ndarray of shape (V, ...), values for each voxel.
        point_to_voxel: np.ndarray of shape (N,), int64 indices mapping points to voxels.
        fill_value: Value to assign for points with invalid mapping (-1).
        
    Returns:
        np.ndarray of shape (N, ...), lifted values for each point.
    """
    N = point_to_voxel.shape[0]
    # Determine output shape based on voxel_values
    # Assuming voxel_values is 1D or 2D for simplicity in this example
    if voxel_values.ndim == 1:
        output_shape = (N,)
    else:
        output_shape = (N,) + voxel_values.shape[1:]
        
    # Initialize with fill value
    lifted_values = np.full(output_shape, fill_value, dtype=voxel_values.dtype)
    
    # Identify valid points
    valid_mask = point_to_voxel >= 0
    
    # Get valid voxel indices
    valid_voxel_indices = point_to_voxel[valid_mask]
    
    # Assign values
    # Note: This assumes voxel_values is 1D. For multi-dimensional features,
    # advanced indexing is required.
    if voxel_values.ndim == 1:
        lifted_values[valid_mask] = voxel_values[valid_voxel_indices]
    else:
        # For multi-dimensional features, use advanced indexing
        # lifted_values[valid_mask] = voxel_values[valid_voxel_indices]
        # This line is a placeholder for the general case as the source 
        # only shows 1D example but mentions features.
        pass
        
    return lifted_values
```

## Subtopics

### 3D Feature Construction via Lifting and Voxelization
This variant combines lifting with voxelization to construct 3D features from multi-view images [S2]. It unprojects pixels to 3D points using camera intrinsics and predicted depth, transforms them to world coordinates, and then voxelize the point cloud [S2]. Features are aggregated within each voxel by averaging the image features of all points in that voxel [S2]. This method is chosen when dense volumetric reasoning from sparse multi-view data is required [S2].

## Operating characteristics
- **Time Complexity:** $O(N)$ for lifting, where $N$ is the number of points, as each point is mapped once [S1, unofficial].
- **Memory Complexity:** $O(N)$ for the output array and $O(V)$ for the voxel values, where $V$ is the number of voxels [S1, unofficial].
- **Strengths:** Preserves original point IDs; handles invalid points gracefully; efficient for dense point clouds [S1, unofficial].
- **Limitations:** Requires pre-computed point-to-voxel mapping; performance depends on the sparsity of the voxel grid [S1, unofficial].
- **When to use:** When reconstructing point-level predictions from voxelized features [S1, unofficial].
- **When not to use:** When the goal is to aggregate data into voxels rather than distribute it back to points [S1, unofficial].

## Background
Voxel Value Lifting is a core operation in point cloud processing pipelines that utilize voxel-based representations [S1, unofficial]. It is commonly used in 3D object detection and segmentation tasks where features are computed in voxel space and need to be mapped back to points for final prediction [S1, unofficial]. The method is implemented in libraries such as `pointcloud-core` [S1, unofficial].

## Sources
[S1] ajavanma/pointcloud-core, https://github.com/ajavanma/pointcloud-core, fetched 2026-10-27 (unofficial)
[S2] VolSplat: Rethinking Feed-Forward 3D Gaussian Splatting with Voxel-Aligned Prediction, https://doi.org/10.1007/978-3-032-37232-1_1, fetched 2026-10-26
