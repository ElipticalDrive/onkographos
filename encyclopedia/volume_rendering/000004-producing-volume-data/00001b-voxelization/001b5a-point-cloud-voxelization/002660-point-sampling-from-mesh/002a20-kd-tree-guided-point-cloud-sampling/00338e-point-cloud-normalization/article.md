# Point Cloud Normalization


## Overview
Point Cloud Normalization standardizes the spatial distribution of 3D points to a canonical coordinate system and scale, ensuring consistent input for downstream voxelization or neural rendering pipelines. It differs from simple bounding-box normalization by offering multiple scaling strategies (average distance, max coordinate, or warp-based) to handle varying point densities and shapes. This method prepares ground truth data for KD-Tree guided sampling by centering objects and constraining their extent.

## How it works
**Inputs:**
- `pts1`: Array of shape $(N_1, 3)$ containing $x, y, z$ coordinates.
- `pts2`: Optional array of shape $(N_2, 3)$ (e.g., for paired data or augmentation).
- `norm_mode`: String specifying normalization strategy. Format: `<norm_type>-<dist_type>` or just `<norm_type>`.
  - `norm_type`: `'avg'` (default).
  - `dist_type`: `'dis'`, `'log1p'`, `'warp-log1p'`.

**Outputs:**
- Normalized `pts1` and `pts2` (if present), scaled and centered.

**Data Structures:**
- Points are stored as float32 arrays.
- Masks for valid points are used to handle invalid entries (converted to zeros before mean calculation).

**Steps:**

1. **Handle Invalid Points:**
   Identify invalid points in `pts1` and `pts2` using valid masks. Replace invalid coordinates with zeros to prevent skewing the mean calculation [S1, unofficial].

2. **Centering (Translation):**
   Calculate the mean coordinate for each axis:
   $$
   x_{mean} = \frac{1}{N} \sum_{i=1}^{N} x_i, \quad y_{mean} = \frac{1}{N} \sum_{i=1}^{N} y_i, \quad z_{mean} = \frac{1}{N} \sum_{i=1}^{N} z_i
   $$
   Subtract the mean from each point:
   $$
   p'_{i} = p_i - [x_{mean}, y_{mean}, z_{mean}]^T
   $$
   If `norm_mode` is `'avg'`, concatenate `pts1` and `pts2` (if `pts2` exists) into a single array `all_pts` to compute global statistics [S1, unofficial].

3. **Distance Calculation:**
   Compute the Euclidean distance of each point from the origin (0,0,0) in the centered coordinate system:
   $$
   all\_dis_i = \| p'_i \| = \sqrt{x'_i^2 + y'_i^2 + z'_i^2}
   $$
   [S1, unofficial]

4. **Scaling and Transformation (Dependent on `dist_type`):**

   - **Case `dis`:**
     Find the maximum absolute value across all normalized coordinates:
     $$
     max\_value = \max(|p'_{i}|)
     $$
     Scale coordinates by dividing by $max\_value \times 2$. This maps the bounding box to $[-0.5, 0.5]$ [S1, unofficial].

   - **Case `log1p`:**
     Transform distances using the log1p function:
     $$
     all\_dis_i = \log(1 + all\_dis_i)
     $$
     Proceed with scaling as in the `dis` case (divide by $max\_value \times 2$) [S1, unofficial].

   - **Case `warp-log1p`:**
     Compute a warp factor for each point:
     $$
     warp\_factor_i = \frac{\log(1 + all\_dis_i)}{\max(10^{-8}, all\_dis_i)}
     $$
     Apply this warp factor to the coordinates of `pts1` and `pts2` [S1, unofficial].
     Translate the coordinates by adding 0.5 to center the object in the $[0.5, 1.5]$ range. This effectively maps the bounding box to $[0, 1]$ and shifts it [S1, unofficial].

5. **Normalization Factor (Only for `avg` mode):**
   If `norm_mode` is `'avg'`, compute a global normalization factor:
   $$
   norm\_factor = \frac{\sum all\_dis}{nnz_1 + nnz_2 + 10^{-8}}
   $$
   where $nnz_1$ and $nnz_2$ are the number of non-zero (valid) points in `pts1` and `pts2` respectively. This factor is used for further scaling if required by the specific implementation context [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def normalize_pointcloud(pts1, pts2=None, norm_mode='avg'):
    """
    Normalize point clouds based on norm_mode.
    
    Args:
        pts1: np.ndarray of shape (N, 3)
        pts2: np.ndarray of shape (M, 3) or None
        norm_mode: str, e.g., 'avg', 'avg-dis', 'avg-log1p', 'avg-warp-log1p'
        
    Returns:
        normalized_pts1, normalized_pts2 (if provided)
    """
    # Parse mode
    # Default is 'avg' for norm_type. 
    # dist_type defaults to 'dis' if not specified or if mode is just 'avg'
    if '-' in norm_mode:
        norm_type, dist_type = norm_mode.split('-')
    else:
        norm_type = norm_mode
        dist_type = 'dis'
        
    # Ensure norm_type is 'avg' as per source logic for this specific method
    # Other norm_types are not described in the provided notes.
    
    # Step 1: Handle invalid points (replace with 0)
    # Assuming valid mask is all True for simplicity if not provided, 
    # but in practice, filter out or zero out invalids.
    # [G] In a real implementation, 'valid_mask' would be passed or derived.
    # Here we assume pts1 and pts2 are already cleaned or we zero out NaNs/Infs.
    pts1 = np.nan_to_num(pts1, nan=0.0, posinf=0.0, neginf=0.0)
    if pts2 is not None:
        pts2 = np.nan_to_num(pts2, nan=0.0, posinf=0.0, neginf=0.0)
        
    # Step 2: Centering
    # Calculate mean for pts1
    x_mean = np.mean(pts1[:, 0])
    y_mean = np.mean(pts1[:, 1])
    z_mean = np.mean(pts1[:, 2])
    
    pts1[:, 0] -= x_mean
    pts1[:, 1] -= y_mean
    pts1[:, 2] -= z_mean
    
    if pts2 is not None:
        # Calculate mean for pts2
        x_mean2 = np.mean(pts2[:, 0])
        y_mean2 = np.mean(pts2[:, 1])
        z_mean2 = np.mean(pts2[:, 2])
        
        pts2[:, 0] -= x_mean2
        pts2[:, 1] -= y_mean2
        pts2[:, 2] -= z_mean2
        
    # If norm_type is 'avg', compute global stats from concatenated points
    if norm_type == 'avg':
        if pts2 is not None:
            all_pts = np.concatenate([pts1, pts2], axis=0)
        else:
            all_pts = pts1
            
        # Step 3: Distance to origin
        all_dis = np.linalg.norm(all_pts, axis=1)
        
        # Step 4: Transform distances based on dist_type
        if dist_type == 'log1p':
            all_dis = np.log1p(all_dis)
        elif dist_type == 'warp-log1p':
            # Compute warp factor
            # Use max(1e-8, all_dis) to avoid division by zero
            warp_factor = np.log1p(all_dis) / np.maximum(1e-8, all_dis)
            
            # Apply warp to pts1 and pts2
            # Note: The source says "Apply warp to pts1 and pts2".
            # Usually, this means scaling the point coordinates by the warp factor.
            # However, the source doesn't explicitly state *how* it's applied 
            # (e.g., point * warp_factor or point * (1 + warp_factor)).
            # Given the context of "warp", and typical implementations, 
            # it often scales the radius. Let's assume point-wise scaling.
            # [G] Since the exact application of warp_factor to coordinates isn't 
            # explicitly detailed in the snippet beyond "Apply warp", we assume 
            # it scales the vector magnitude. A common way is:
            # p_new = p * (warp_factor / original_dis) * original_dis ? 
            # Or simply p_new = p * warp_factor? 
            # Looking at the formula: warp_factor = log(1+d)/d. 
            # If we multiply p by warp_factor, we get p * log(1+d)/d.
            # This seems to be the intended operation.
            pts1 = pts1 * warp_factor[:len(pts1)].reshape(-1, 1)
            pts2 = pts2 * warp_factor[len(pts1):].reshape(-1, 1)
            
            # Translate by 0.5
            pts1 += 0.5
            pts2 += 0.5
            
            # Return early as scaling step below is for 'dis' and 'log1p'
            return pts1, pts2
            
        # For 'dis' and 'log1p', we still need to scale the coordinates
        # Find max absolute value in normalized coordinates (before warp/log transform of coords)
        # Note: The source says "Find the maximum absolute value in the normalized coordinates"
        # This refers to the centered coordinates (pts1/pts2) before scaling.
        max_value = np.max(np.abs(np.concatenate([pts1, pts2], axis=0)))
        
        # Scale coordinates
        scale_factor = max_value * 2
        pts1 /= scale_factor
        pts2 /= scale_factor
        
        # Step 5: Compute norm factor (for logging or further use)
        nnz1 = len(pts1) # Assuming all are valid after nan_to_num
        nnz2 = len(pts2)
        norm_factor = np.sum(all_dis) / (nnz1 + nnz2 + 1e-8)
        
        return pts1, pts2
    else:
        # If norm_type is not 'avg', the source doesn't specify behavior.
        # Assuming 'avg' is the only supported mode in this context.
        return pts1, pts2
```

**Parameters:**
- `norm_mode`: String. Options: `'avg'`, `'avg-dis'`, `'avg-log1p'`, `'avg-warp-log1p'`. Default: `'avg'` (implies `dis` scaling) [S1, unofficial].

**Edge Cases:**
- **Division by Zero:** When computing `warp_factor`, if `all_dis` is 0, use $\max(10^{-8}, all\_dis)$ to prevent division by zero [S1, unofficial].
- **Invalid Points:** Points with NaN or Inf values must be handled (converted to 0) before mean calculation to avoid propagating invalid statistics [S1, unofficial].

## Subtopics

### Principal Axes Alignment
Computes a 4x4 SE(3) transform that aligns a point cloud to its principal axes of covariance, centering it at the median. It computes the centroid as the median of the point cloud, translates the cloud to the centroid, computes the covariance matrix, and extracts eigenvectors. Eigenvectors are sorted by eigenvalues in descending order so the z-axis corresponds to the principal axis with the smallest eigenvalue. A rotation matrix is formed from the eigenvectors, and an SE(3) transform is constructed. If the determinant of the eigenvector matrix is negative, the sign of the first eigenvector is flipped to ensure a right-handed coordinate system [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ for mean and distance calculations, $O(N)$ for covariance matrix computation (if used in variants), $O(1)$ for eigendecomposition of a 3x3 matrix. Overall linear with respect to the number of points $N$.
- **Memory Complexity:** $O(N)$ to store the point cloud and intermediate arrays.
- **Strengths:** Robust to scale variations; `warp-log1p` handles sparse or dense regions differently, potentially preserving local structure better than uniform scaling.
- **Limitations:** `avg` mode assumes a roughly symmetric distribution around the mean. `warp-log1p` is more complex and may introduce non-linear distortions.
- **When to use:** When preparing point clouds for voxelization or neural rendering where consistent scale and centering are critical. Use `warp-log1p` if the point cloud has significant variation in density or extent.

## Background
Originating from implementations in neural rendering pipelines like InstantSplat and mrGSplat, point cloud normalization is a standard preprocessing step. It ensures that input data is in a canonical space, facilitating stable training and evaluation of 3D reconstruction models. The specific variants (`avg`, `log1p`, `warp-log1p`) are designed to handle different statistical properties of the input point clouds [S1, S3, unofficial].

## Sources
[S1] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S2] MasonAndrewHarrison/Point-Cloud-Voxelizer, https://github.com/MasonAndrewHarrison/Point-Cloud-Voxelizer, fetched 2026-10-27 (unofficial)
[S3] meshroomHub/mrGSplat, https://github.com/meshroomHub/mrGSplat, fetched 2026-10-27 (unofficial)
