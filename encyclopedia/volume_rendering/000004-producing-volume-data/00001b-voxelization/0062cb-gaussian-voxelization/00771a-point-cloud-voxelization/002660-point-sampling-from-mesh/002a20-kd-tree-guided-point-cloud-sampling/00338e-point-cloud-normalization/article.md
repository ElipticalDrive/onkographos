# Point Cloud Normalization


## Overview
Point Cloud Normalization standardizes point cloud coordinates to a canonical scale and center for downstream processing such as voxelization [S1, unofficial]. It differs from rotational alignment methods by applying statistical centering and bounding-box scaling rather than covariance-based rotation [S2, unofficial]. The method supports multiple normalization modes ('avg', 'dis', 'log1p', 'warp-log1p') [S1, unofficial].

## How it works
**Inputs:**
- `pts1`: Array of shape $(N, 3)$ containing point coordinates.
- `pts2`: Optional array of shape $(M, 3)$ for joint normalization.
- `valid_mask`: Optional boolean array of shape $(N,)$ indicating valid points in `pts1`.
- `norm_mode`: String specifying the normalization strategy. Options: `'avg'`, `'dis'`, `'log1p'`, `'warp-log1p'` [S1, unofficial].

**Outputs:**
- Normalized `pts1` and `pts2` (if provided) with coordinates scaled to a canonical range [S1, unofficial].

**Data Structures:**
- Points are stored as floating-point arrays [G]. No specific bit-packing is defined in the sources.

**Steps:**

1. **Handle Invalid Points:** If `norm_mode` is `'avg'`, replace invalid points in `pts1` with zeros using `valid_mask` [S1, unofficial].
2. **Centering:** For each point cloud, calculate the mean of x, y, and z coordinates, then subtract the mean from each coordinate [S2, unofficial].
3. **Scaling:** Find the maximum absolute value across all normalized coordinates, then scale the coordinates by dividing by `max_value * 2` [S2, unofficial].
4. **Translation:** Translate the coordinates by adding 0.5 to center the object in the $[0.5, 1.5]$ range [S2, unofficial].
5. **Distance & Norm Factor (for 'avg' mode):** Concatenate `pts1` and `pts2` (if present) into `all_pts`. Compute the Euclidean distance to the origin: $all\_dis = \|all\_pts\|$ [S1, unofficial]. Compute the norm factor: $norm\_factor = \frac{\sum all\_dis}{nnz1 + nnz2 + 10^{-8}}$, where $nnz1$ and $nnz2$ are the counts of valid points in `pts1` and `pts2` respectively [S1, unofficial].
6. **Distance Transformation (for 'log1p' mode):** Transform distances: $all\_dis = \log(1 + all\_dis)$ [S1, unofficial].
7. **Warp Application (for 'warp-log1p' mode):** Compute warp factor $warp\_factor = \frac{\log(1 + all\_dis)}{\max(10^{-8}, all\_dis)}$ and apply it to `pts1` and `pts2` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def normalize_pointcloud(pts1, pts2=None, valid_mask=None, norm_mode='avg'):
    """
    Normalize point clouds based on norm_mode.
    
    Args:
        pts1: np.ndarray of shape (N, 3)
        pts2: np.ndarray of shape (M, 3) or None
        valid_mask: np.ndarray of shape (N,) boolean, or None
        norm_mode: str, one of 'avg', 'dis', 'log1p', 'warp-log1p'
        
    Returns:
        normalized_pts1, normalized_pts2 (or None), norm_factor (or None)
    """
    # Split norm_mode to extract dis_mode if present
    parts = norm_mode.split('-')
    mode = parts[0]
    dis_mode = parts[1] if len(parts) > 1 else None

    # Step 1: Handle invalid points for 'avg' mode
    if mode == 'avg':
        if valid_mask is not None:
            pts1 = pts1.copy()
            pts1[~valid_mask] = 0.0

    # Step 2: Centering
    def center(pts):
        return pts - np.mean(pts, axis=0)

    pts1_c = center(pts1)
    pts2_c = center(pts2) if pts2 is not None else None

    # Step 3: Scaling
    max_val1 = np.max(np.abs(pts1_c))
    max_val2 = np.max(np.abs(pts2_c)) if pts2_c is not None else 0.0
    max_value = max(max_val1, max_val2)
    pts1_s = pts1_c / (max_value * 2.0)
    pts2_s = pts2_c / (max_value * 2.0) if pts2_c is not None else None

    # Step 4: Translation
    pts1_t = pts1_s + 0.5
    pts2_t = pts2_s + 0.5 if pts2_s is not None else None

    # Step 5 & 6 & 7: Mode-specific distance/warp processing
    all_dis = None
    norm_factor = None
    nnz1 = np.sum(valid_mask) if valid_mask is not None else pts1.shape[0]
    nnz2 = pts2.shape[0] if pts2 is not None else 0

    if mode == 'avg':
        all_pts = np.concatenate([pts1_t, pts2_t], axis=0) if pts2_t is not None else pts1_t
        all_dis = np.linalg.norm(all_pts, axis=1)
        norm_factor = np.sum(all_dis) / (nnz1 + nnz2 + 1e-8)

    if dis_mode == 'log1p':
        if all_dis is None:
            all_pts = np.concatenate([pts1_t, pts2_t], axis=0) if pts2_t is not None else pts1_t
            all_dis = np.linalg.norm(all_pts, axis=1)
        all_dis = np.log(1.0 + all_dis)

    if dis_mode == 'warp-log1p':
        if all_dis is None:
            all_pts = np.concatenate([pts1_t, pts2_t], axis=0) if pts2_t is not None else pts1_t
            all_dis = np.linalg.norm(all_pts, axis=1)
        warp_factor = np.log(1.0 + all_dis) / np.maximum(1e-8, all_dis)
        n1 = pts1_t.shape[0]
        pts1_t = pts1_t * warp_factor[:n1, np.newaxis]
        if pts2_t is not None:
            n2 = pts2_t.shape[0]
            pts2_t = pts2_t * warp_factor[n1:n1+n2, np.newaxis]

    return pts1_t, pts2_t, norm_factor
```

**Parameters:**
- `norm_mode`: String. Options: `'avg'`, `'dis'`, `'log1p'`, `'warp-log1p'` [S1, unofficial].

**Edge Cases:**
- **Empty Point Clouds:** Division by zero in mean calculation or scaling. Guard against $N=0$ [G].
- **Invalid Points:** Handled by `valid_mask` in `'avg'` mode [S1, unofficial].
- **Zero Distance:** Warp factor calculation uses $\max(10^{-8}, all\_dis)$ to avoid division by zero [S1, unofficial].

## Subtopics

### [Principal Axes Alignment](11372-principal-axes-alignment.md)
Computes a 4x4 SE(3) transform that aligns a point cloud to its principal axes of covariance while centering it at the median. It serves as the final normalization step in the point cloud preprocessing pipeline, ensuring that subsequent geometric operations operate on a standardized coordinate frame. It differs from sibling methods by explicitly decoupling scale initialization from rotational initialization and by using the median for centering rather than the mean, which provides robustness against outliers.

### Mesh Extraction and Point Cloud Normalization
A post-processing pipeline that converts a 3D representation to a mesh and normalizes the resulting point cloud for geometric evaluation [S3]. It extracts meshes using the marching cubes algorithm, samples 2048 points uniformly on the mesh faces, centers the point cloud at the origin, and applies anisotropic scaling to bound the extent to $[-1, 1]$ [S3].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths and Limitations:** Not given in the sources.
- **When to Use and When Not to Use:** Not given in the sources.

## Background
Origin and history: Not given in the sources. It is used to preprocess ground truth point clouds for the KD-Tree Guided Point Cloud Sampling pipeline [S1, unofficial].

## Sources
[S1] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S2] MasonAndrewHarrison/Point-Cloud-Voxelizer, https://github.com/MasonAndrewHarrison/Point-Cloud-Voxelizer, fetched 2026-10-27 (unofficial)
[S3] DiffRF: Rendering-Guided 3D Radiance Field Diffusion, https://doi.org/10.1109/cvpr52729.2023.00421, fetched 2026-10-26
