# Point Cloud Simplification via Voxel-based Probabilistic Sub-sampling

Also called: Uniform voxel sampling, Voxel-based point cloud downsampling

## Overview
This method reduces point cloud density by grouping points into a 3D grid and probabilistically selecting a fixed number of points per voxel [S1, unofficial]. It builds on [Hard Voxelization](5345-hard-voxelization.md) by replacing deterministic aggregation (e.g., averaging or storing all points) with a stochastic selection process that preserves the statistical distribution of points while enforcing a strict capacity limit per voxel [S1, unofficial]. Unlike siblings that may use fixed rules or multiscale approaches, this method ensures that sparse voxels are not discarded entirely but are sampled proportionally, making it suitable for scenes with varying point densities where memory constraints require a predictable output size [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A point cloud represented as a set of $N$ points, each with 3D coordinates $(x, y, z)$ and optional attributes (e.g., RGB color). The input is typically stored in a format such as `.ply` ASCII, where each vertex row contains 5 space-separated values: `x y z r g b` [S1, unofficial].
- **Output:** A simplified point cloud with $N' \le N$ points, where each voxel contains at most `num_in_each_voxel` points. The output retains the original format and attributes of the selected points [S1, unofficial].

**Data Structures:**
1.  **Point Array:** A 2D array of shape $(N, D)$, where $D$ is the number of attributes (e.g., 6 for XYZ + RGB).
2.  **Voxel Index Array:** A 1D integer array of length $N$, storing the linearized voxel index for each point.
3.  **Probability Array:** A 1D float array of length $N$, storing the selection probability for each point.
4.  **Grid Dimensions:** Three integers `num_x`, `num_y`, `num_z` defining the grid size along each axis.

**Parameters:**
-   `side_len`: Voxel side length in meters. Typical value: 0.01 m [S1, unofficial]. Other examples use 0.02, 0.03, 0.05, or 0.1 m [S1, unofficial].
-   `num_in_each_voxel`: Predefined maximum number of points to retain per voxel. Typical value: 20 [S1, unofficial]. Examples show 30, 15, 5, 3, 2, or 1 [S1, unofficial].

**Steps:**
1.  **Compute Bounding Box:** Determine the minimum and maximum coordinates for $x, y, z$ across all points: $x_{min}, x_{max}, y_{min}, y_{max}, z_{min}, z_{max}$ [S1, unofficial].
2.  **Calculate Grid Dimensions:** Compute the number of voxels along each axis using rounding:
    $$
    \text{num}_x = \max(1, \lfloor \frac{x_{max} - x_{min}}{\text{side\_len}} \rfloor + 1)
    $$
    Similarly for `num_y` and `num_z` [S1, unofficial]. Note: The source code uses `np.rint` (round to nearest integer) on the coordinate differences before division, then adds 1. Specifically: `num_x = np.max(np.rint((x - min_x) / self.side_len)) + 1` [S1, unofficial].
3.  **Compute Voxel Indices:** For each point $i$ with coordinates $(x_i, y_i, z_i)$, compute the 3D voxel index $(i_x, i_y, i_z)$ and linearize it:
    $$
    i_x = \text{round}\left(\frac{x_i - x_{min}}{\text{side\_len}}\right), \quad
    i_y = \text{round}\left(\frac{y_i - y_{min}}{\text{side\_len}}\right), \quad
    i_z = \text{round}\left(\frac{z_i - z_{min}}{\text{side\_len}}\right)
    $$
    $$
    \text{index}_i = i_x + i_y \cdot \text{num}_x + i_z \cdot \text{num}_x \cdot \text{num}_y
    $$
    [S1, unofficial]. Points falling outside the grid bounds (if any) should be handled, though the source assumes all points fit within the computed bounds [S1, unofficial].
4.  **Sort Points:** Sort the points based on their voxel indices to group points belonging to the same voxel contiguously. Store the permutation indices to reorder the point attributes accordingly [S1, unofficial].
5.  **Initialize Probabilities:** Create an array `select_probabilities` of length $N$, initialized to 0 [S1, unofficial].
6.  **Assign Selection Probabilities:** Iterate through the sorted points to identify ranges belonging to each voxel. For a voxel containing $C$ points, the selection probability for each point in that voxel is:
    $$
    p = \frac{\min(C, \text{num\_in\_each\_voxel})}{C}
    $$
    Assign this probability to all points in the voxel's range in `select_probabilities` [S1, unofficial].
7.  **Generate Random Numbers:** Generate $N$ random numbers from a uniform distribution $U(0, 1)$ [S1, unofficial].
8.  **Select Points:** For each point $i$, if `select_probabilities[i]` > `random[i]`, include the point in the output. Copy the point's attributes to the output array and increment the output counter [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def voxel_probabilistic_subsampling(points, side_len, num_in_each_voxel):
    """
    points: numpy array of shape (N, D), where D >= 3 (XYZ + attributes)
    side_len: float, voxel size
    num_in_each_voxel: int, max points per voxel
    """
    N = points.shape[0]
    coords = points[:, :3]
    
    # Step 1: Compute bounding box
    min_x, min_y, min_z = np.min(coords, axis=0)
    max_x, max_y, max_z = np.max(coords, axis=0)
    
    # Step 2: Calculate grid dimensions
    # Using np.rint as per source code
    num_x = int(np.max(np.rint((coords[:, 0] - min_x) / side_len))) + 1
    num_y = int(np.max(np.rint((coords[:, 1] - min_y) / side_len))) + 1
    num_z = int(np.max(np.rint((coords[:, 2] - min_z) / side_len))) + 1
    
    # Step 3: Compute voxel indices
    ix = np.rint((coords[:, 0] - min_x) / side_len).astype(int)
    iy = np.rint((coords[:, 1] - min_y) / side_len).astype(int)
    iz = np.rint((coords[:, 2] - min_z) / side_len).astype(int)
    
    indices = ix + iy * num_x + iz * num_x * num_y
    
    # Step 4: Sort points by voxel index
    sort_idx = np.argsort(indices)
    sorted_indices = indices[sort_idx]
    sorted_points = points[sort_idx]
    
    # Step 5 & 6: Assign probabilities
    select_probabilities = np.zeros(N)
    cnt = 0
    while cnt < N:
        # Find the end of the current voxel group
        # Find the first index where the voxel index changes
        if cnt + 1 >= N:
            end = N
        else:
            # Find the first index > cnt with a different index
            # Since sorted, we can scan forward
            current_idx = sorted_indices[cnt]
            end = cnt
            while end < N and sorted_indices[end] == current_idx:
                end += 1
        
        count_in_voxel = end - cnt
        # Probability is min(count, num_in_each_voxel) / count
        prob = min(count_in_voxel, num_in_each_voxel) / count_in_voxel
        select_probabilities[cnt:end] = prob
        cnt = end
        
    # Step 7: Generate random numbers
    random_vals = np.random.uniform(0, 1, N)
    
    # Step 8: Select points
    selected_mask = select_probabilities > random_vals
    selected_points = sorted_points[selected_mask]
    
    # Return points in original order? The source copies to output array incrementally.
    # The output order is the order of selection in the sorted array.
    # If original order is needed, map back using sort_idx.
    # However, the source code simply copies to `position_color_after[cnt, :]`
    # implying the output is in the sorted/selected order.
    # We will return the selected points.
    return selected_points
```

**Edge Cases and Pitfalls:**
-   **Empty Voxels:** If a voxel has 0 points, it contributes nothing to the output. The probability calculation is only applied to voxels with at least one point [S1, unofficial].
-   **Last Voxel Group:** The loop handling point groups must correctly identify the end of the last voxel group. The source uses an `else` branch or boundary check to ensure `cnt_r` (end index) does not exceed $N$ [S1, unofficial].
-   **Memory Usage:** Large scenes can cause out-of-memory errors due to excessive voxel count or large point arrays. The source suggests dividing the scene into smaller parts and processing them separately [S1, unofficial].
-   **Determinism:** The method is non-deterministic due to random sampling. For reproducible results, a fixed random seed must be set [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** Dominated by sorting, $O(N \log N)$ for $N$ points [S1, unofficial]. The subsequent linear scan is $O(N)$.
-   **Memory Complexity:** $O(N)$ for storing position, color, index, and probability arrays [S1, unofficial].
-   **Strengths:** Reduces memory footprint significantly while preserving point distribution statistics. Simple to implement.
-   **Limitations:** Non-deterministic output. May remove points from sparse regions if the probability is low, potentially losing fine details. Performance degrades with very large scenes unless chunked [S1, unofficial].
-   **When to Use:** When memory is constrained and a uniform reduction in point density is acceptable. Suitable for preprocessing for NeRF or other neural rendering methods that are robust to point density variations [S1, unofficial].
-   **When Not to Use:** When deterministic output is required, or when preserving every point in sparse regions is critical.

## Background
This method is described in the context of "Rendering via Multi-plane NeRF" by Dongmei Ma, Juan Cao, and Zhonggui Chen (CGI 2023) [S1, unofficial]. It is used as a preprocessing step to simplify point clouds before they are used as input for neural rendering networks, specifically to manage memory and computational load [S1, unofficial]. The technique is implemented in the GitHub repository `Mayxmu/PCMP_NeRF` [S1, unofficial].

## Sources
[S1] https://github.com/Mayxmu/PCMP-NeRF, https://github.com/Mayxmu/PCMP_NeRF, fetched 2026-10-30 (unofficial)
