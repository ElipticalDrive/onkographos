# Grid Pooling (Max Pooling over Points)
Also called: Point-to-Grid Max Pooling

## Overview
Grid Pooling aggregates point cloud features into a fixed-size 3D voxel grid by performing max-pooling within each grid cell [S1, unofficial]. It maps a set of $N$ points with $C$-dimensional features to a $W \times H \times D$ grid where each cell stores the maximum feature value among points falling into that cell [S1, unofficial]. It differs from its sibling [Hard Voxelization (Deterministic)](3321-hard-voxelization-deterministic.md) by explicitly tracking the index of the point that achieved the maximum value to enable differentiable backpropagation [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: `points` tensor of shape $(N, 3)$ containing 3D coordinates [S1, unofficial].
- Input: `feat_points` tensor of shape $(N, C)$ containing features for each point [S1, unofficial].
- Input: `shape` tensor of shape $(3)$ containing grid dimensions $(W, H, D)$ [S1, unofficial].
- Input: `grid_size` float defining the physical size of each grid cell (typically 1.0) [S1, unofficial].
- Output: `feat_cell` tensor of shape $(W \times H \times D, C)$ storing the maximum feature values [S1, unofficial].
- Output: `indices` tensor of shape $(W \times H \times D, C)$ storing the global point index that achieved the max value for each cell and feature channel [S1, unofficial].

**Data Structures:**
- `feat_cell`: Float tensor, initialized to negative infinity (or zero if features are guaranteed positive, but negative infinity is safer for general max-pooling) [S1, unofficial].
- `indices`: Long tensor, initialized to -1 to indicate empty cells or invalid states [S1, unofficial].

**Parameters:**
- `grid_size`: Constant float, typically 1.0 [S1, unofficial].
- `eps`: Constant float, typically $10^{-8}$, declared for numerical stability in comparisons or divisions, though not explicitly used in the provided CPU implementation snippet [S1, unofficial].

**Algorithm Steps:**
1. **Initialization**: Initialize `feat_cell` with negative infinity and `indices` with -1 for all cells $(i, j, k)$ where $0 \le i < W, 0 \le j < H, 0 \le k < D$ [S1, unofficial].
2. **Forward Pass**:
   For each grid cell $(i, j, k)$:
   a. Identify all points $p$ such that their coordinates $(x_p, y_p, z_p)$ satisfy:
      $$ i \cdot \text{grid\_size} \le x_p < (i+1) \cdot \text{grid\_size} $$
      $$ j \cdot \text{grid\_size} \le y_p < (j+1) \cdot \text{grid\_size} $$
      $$ k \cdot \text{grid\_size} \le z_p < (k+1) \cdot \text{grid\_size} $$
      Let $P_{cell}$ be the set of indices of these points [S1, unofficial].
   b. If $P_{cell}$ is empty, skip to the next cell [S1, unofficial].
   c. For each feature dimension $c \in [0, C-1]$:
      i. Find the point $p^* \in P_{cell}$ that maximizes `feat_points[p, c]` [S1, unofficial].
      ii. Set `feat_cell[i*H*D + j*D + k, c] = feat_points[p*, c]` [S1, unofficial].
      iii. Set `indices[i*H*D + j*D + k, c] = p*` [S1, unofficial].
3. **Backward Pass**:
   For each grid cell $(i, j, k)$ and feature dimension $c$:
   a. Retrieve the stored index $p = \text{indices}[i \cdot H \cdot D + j \cdot D + k, c]$ [S1, unofficial].
   b. If $p < 0$ (empty cell), skip [S1, unofficial].
   c. Propagate the gradient from the output gradient tensor `grad_output` to the input feature gradient tensor `grad_feat_points`:
      $$ \text{grad\_feat\_points}[p, c] \leftarrow \text{grad\_output}[i \cdot H \cdot D + j \cdot D + k, c] $$
      Note: The reference implementation uses an assignment operation (`set2d`), which overwrites the value [S1, unofficial]. If multiple cells select the same point as their max (due to duplicate points or specific data distributions), only the last processed cell's gradient is retained [S1, unofficial]. Standard max pooling backpropagation should sum gradients for all cells where the point was the max [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def grid_pooling_forward(points, feat_points, shape, grid_size=1.0):
    """
    points: Tensor of shape (N, 3)
    feat_points: Tensor of shape (N, C)
    shape: Tuple (W, H, D)
    grid_size: Float
    Returns: feat_cell (W*H*D, C), indices (W*H*D, C)
    """
    N, C = feat_points.shape
    W, H, D = shape
    num_cells = W * H * D
    
    # Initialize outputs
    feat_cell = torch.full((num_cells, C), float('-inf'), dtype=feat_points.dtype, device=feat_points.device)
    indices = torch.full((num_cells, C), -1, dtype=torch.long, device=points.device)
    
    for i in range(W):
        x_min = i * grid_size
        x_max = (i + 1) * grid_size
        for j in range(H):
            y_min = j * grid_size
            y_max = (j + 1) * grid_size
            for k in range(D):
                z_min = k * grid_size
                z_max = (k + 1) * grid_size
                
                cell_idx = i * H * D + j * D + k
                
                # Find points in this cell
                in_x = (points[:, 0] >= x_min) & (points[:, 0] < x_max)
                in_y = (points[:, 1] >= y_min) & (points[:, 1] < y_max)
                in_z = (points[:, 2] >= z_min) & (points[:, 2] < z_max)
                mask = in_x & in_y & in_z
                
                point_indices_in_cell = torch.nonzero(mask, as_tuple=False).squeeze(1)
                
                if point_indices_in_cell.numel() == 0:
                    continue
                
                # Get features for these points
                cell_features = feat_points[point_indices_in_cell]
                
                # Max pooling over the point dimension (dim 0)
                max_vals, max_indices_local = torch.max(cell_features, dim=0)
                
                # Convert local indices to global point indices
                global_indices = point_indices_in_cell[max_indices_local]
                
                # Update output tensors
                feat_cell[cell_idx, :] = max_vals
                indices[cell_idx, :] = global_indices
                
    return feat_cell, indices

def grid_pooling_backward(grad_output, indices, N):
    """
    grad_output: Tensor of shape (W*H*D, C)
    indices: Tensor of shape (W*H*D, C)
    N: int, number of points
    Returns: grad_feat_points (N, C)
    """
    grad_feat_points = torch.zeros((N, grad_output.shape[1]), dtype=grad_output.dtype, device=grad_output.device)
    
    # Flatten for easier iteration
    indices_flat = indices.view(-1)
    grad_flat = grad_output.view(-1)
    
    # The reference implementation uses set2d (overwrite).
    # We replicate the overwrite behavior exactly as described in the source.
    valid_mask = indices_flat >= 0
    valid_indices = indices_flat[valid_mask]
    valid_grads = grad_flat[valid_mask]
    
    # Overwrite: grad_feat_points[ind] = grad
    for idx, grad in zip(valid_indices, valid_grads):
        grad_feat_points[idx] = grad
        
    return grad_feat_points
```

**Edge Cases and Pitfalls:**
- **Empty Cells**: If no point falls into a grid cell, `indices` remains -1 [S1, unofficial]. The backward pass must check for $p < 0$ to prevent invalid memory access [S1, unofficial].
- **Gradient Assignment Pitfall**: The reference C implementation uses `THFloatTensor_set2d`, which overwrites the gradient value [S1, unofficial]. If multiple cells select the same point as their max, only the last processed cell's gradient is retained [S1, unofficial]. Standard max pooling backpropagation should sum gradients for all cells where the point was the max [S1, unofficial].
- **Precision**: The `eps` parameter is declared but not used in the provided CPU snippet [S1, unofficial]. Numerical stability should be ensured in comparisons if features are very close [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(W \cdot H \cdot D \cdot P_{avg})$ for the forward pass, where $P_{avg}$ is the average number of points per cell [S1, unofficial]. The backward pass is $O(W \cdot H \cdot D \cdot C)$ [S1, unofficial].
- **Memory Complexity**: $O(W \cdot H \cdot D \cdot C)$ for `feat_cell` and `indices` [S1, unofficial].
- **Strengths**: Differentiable, allowing gradients to flow back to point features [S1, unofficial]. Efficient for sparse point clouds mapped to dense grids [S1, unofficial].
- **Limitations**: The "overwrite" pitfall in the reference implementation can lead to incorrect gradients if not handled [S1, unofficial]. Performance degrades if points are densely clustered in few cells, increasing $P_{avg}$ [S1, unofficial].
- **When to use**: When point cloud features need to be aggregated into a voxel grid for subsequent 3D CNN or grid-based processing, and differentiability is required [S1, unofficial].
- **When not to use**: If memory is constrained and the grid is very large, or if non-differentiable aggregation is sufficient [S1, unofficial].

## Background
Grid Pooling is used in deep learning architectures for 3D point cloud processing, specifically in methods that combine point-based features with voxel-based representations [S1, unofficial].

## Sources
[S1] yiyiliao/deep_marching_cubes, https://github.com/yiyiliao/deep_marching_cubes, fetched 2026-10-26 (unofficial)
