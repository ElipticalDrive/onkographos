# HexPlaneField


## Overview
HexPlaneField is a neural field representation designed for dynamic scenes, utilizing a set of 2D and 3D planes (grids) indexed by coordinate combinations to support multi-scale resolutions and time-dependent features [S1, unofficial]. It distinguishes itself from standard hash grid methods by explicitly decomposing the 4D space (x, y, z, t) into lower-dimensional planar and volumetric features, enabling efficient real-time rendering of dynamic content [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A batch of 3D spatial points $\mathbf{pts} \in \mathbb{R}^{N \times 3}$ and corresponding timestamps $\mathbf{t} \in \mathbb{R}^{N}$.
*   **Output:** A feature vector $\mathbf{f} \in \mathbb{R}^{N \times D}$, where $D$ is the output dimension defined by `output_coordinate_dim`.

**Data Structures:**
The core data structure is `self.grids`, a `ModuleList` of `ParameterList`s.
*   **Structure:** `self.grids[scale][plane_index]` stores the learnable parameters.
*   **Layout:**
    *   For 2D planes (e.g., $xy, xt$): Shape is $[1, \text{out\_dim}, \text{reso}_x, \text{reso}_y]$.
    *   For 3D cubes (e.g., $xyz$): Shape is $[1, \text{out\_dim}, \text{reso}_x, \text{reso}_y, \text{reso}_z]$.
*   **Coordinate Combinations:** For 4D input $(x,y,z,t)$, grids correspond to combinations such as $xy, xz, yz, xt, yt, zt$ and potentially $xyz$ [S1, unofficial].

**Parameters:**
*   `bounds`: Defines the Axis-Aligned Bounding Box (AABB) as $[[\text{bounds}, \text{bounds}, \text{bounds}], [-\text{bounds}, -\text{bounds}, -\text{bounds}]]$ [S1, unofficial].
*   `planeconfig`: Dictionary containing `grid_dimensions`, `input_coordinate_dim`, `output_coordinate_dim`, and `resolution` [S1, unofficial].
*   `multires`: List of multi-scale resolution multipliers applied to spatial planes [S1, unofficial].

**Algorithm Steps:**
1.  **Normalization:** Normalize input points $\mathbf{pts}$ to $[-1, 1]$ range using the AABB:
    $$ \mathbf{pts}_{\text{norm}} = \frac{2.0 (\mathbf{pts} - \text{aabb}[0])}{\text{aabb}[1] - \text{aabb}[0]} - 1.0 $$
    where $\text{aabb}[0]$ is the minimum corner and $\text{aabb}[1]$ is the maximum corner [S1, unofficial].
2.  **Concatenation:** Concatenate the normalized spatial coordinates with the timestamps:
    $$ \mathbf{pts}_{\text{cat}} = \text{concat}(\mathbf{pts}_{\text{norm}}, \mathbf{t}) $$
    [S1, unofficial].
3.  **Grid Sampling:** For each scale and each plane/cube combination:
    *   Extract the relevant 2D or 3D coordinates from $\mathbf{pts}_{\text{cat}}$.
    *   Use `grid_sample_wrapper` to interpolate features from the corresponding grid parameter using bilinear/trilinear interpolation.
    *   *Note:* `grid_sample_wrapper` only supports grid dimensions 2 or 3 [S1, unofficial].
4.  **Aggregation:** Concatenate the interpolated features from all planes and cubes across all scales.
5.  **Output:** Return the aggregated feature tensor. If the feature tensor is empty (length < 1), return a zero tensor of shape $(0, 1)$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def hex_plane_field_forward(pts, timestamps, grids, bounds, planeconfig):
    """
    pts: (N, 3) spatial coordinates
    timestamps: (N,) time values
    grids: ModuleList of ParameterList, shape [scale][plane] -> [1, out_dim, res_x, res_y, ...]
    bounds: float, half-extent of the scene
    planeconfig: dict with 'output_coordinate_dim'
    """
    # 1. Normalize points to [-1, 1]
    aabb_min = torch.tensor([-bounds, -bounds, -bounds], device=pts.device)
    aabb_max = torch.tensor([bounds, bounds, bounds], device=pts.device)
    pts_norm = 2.0 * (pts - aabb_min) / (aabb_max - aabb_min) - 1.0
    
    # 2. Concatenate timestamps
    # timestamps needs to be shaped (N, 1) for concatenation
    pts_cat = torch.cat([pts_norm, timestamps.unsqueeze(1)], dim=-1) # (N, 4)
    
    all_features = []
    
    # 3. Iterate through scales and planes
    # Note: The specific mapping of planes (xy, xt, etc.) depends on implementation details not fully specified in sources.
    # We assume a standard set of 2D planes and optional 3D cubes for 4D input.
    plane_combinations = [
        (0, 1), (0, 2), (1, 2), # xy, xz, yz
        (0, 3), (1, 3), (2, 3)  # xt, yt, zt
    ]
    
    for scale_idx, scale_grids in enumerate(grids):
        scale_features = []
        
        # Process 2D planes
        for i, (dim1, dim2) in enumerate(plane_combinations):
            # Extract coordinates for this plane
            coords_2d = pts_cat[:, [dim1, dim2]] # (N, 2)
            
            # Get grid for this plane at this scale
            # Assuming grids are ordered consistently with plane_combinations
            grid = scale_grids[i] # Shape: [1, out_dim, res1, res2]
            
            # Reshape for grid_sample: (N, 1, 1, 2) -> (1, N, 1, 2) for batched sampling?
            # Standard grid_sample expects grid of shape (N, H_out, W_out, 2)
            # Here we sample at specific points. We create a grid of normalized coordinates.
            # Since pts_norm is already in [-1, 1], we can use it directly as the grid.
            
            # grid_sample requires grid shape (N, H_out, W_out, 2)
            # We treat each point as a 1x1 grid cell
            grid_input = coords_2d.unsqueeze(1).unsqueeze(1) # (N, 1, 1, 2)
            
            # Interpolate
            # grid: (1, C, H, W), input: (N, 1, 1, 2)
            # output: (N, C, 1, 1)
            feat = F.grid_sample(grid, grid_input, mode='bilinear', padding_mode='border', align_corners=True)
            feat = feat.squeeze(-1).squeeze(-1) # (N, C)
            scale_features.append(feat)
            
        # Process 3D cubes (if configured)
        # Assuming one 3D cube grid per scale, typically for xyz
        if len(scale_grids) > len(plane_combinations):
            grid_3d = scale_grids[len(plane_combinations)] # Shape: [1, out_dim, res_x, res_y, res_z]
            coords_3d = pts_cat[:, :3] # (N, 3)
            grid_input_3d = coords_3d.unsqueeze(1).unsqueeze(1) # (N, 1, 1, 3)
            feat_3d = F.grid_sample(grid_3d, grid_input_3d, mode='trilinear', padding_mode='border', align_corners=True)
            feat_3d = feat_3d.squeeze(-1).squeeze(-1) # (N, C)
            scale_features.append(feat_3d)
            
        all_features.append(torch.cat(scale_features, dim=-1))
        
    # 4. Concatenate all scales
    if not all_features:
        return torch.zeros((pts.shape[0], 1), device=pts.device)
        
    final_features = torch.cat(all_features, dim=-1)
    return final_features
```

**Edge Cases and Pitfalls:**
*   **Grid Dimension Mismatch:** The underlying `grid_sample_wrapper` only supports 2D and 3D data. If a grid dimension other than 2 or 3 is requested, a `NotImplementedError` is raised [S1, unofficial].
*   **Empty Features:** If the interpolated features tensor is empty (length < 1), the function returns a zero tensor of shape $(0, 1)$ to prevent downstream errors [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of points $N$ and the number of planes/scales. Interpolation is $O(1)$ per point per plane.
*   **Memory Complexity:** Proportional to the sum of the sizes of all 2D and 3D grids across all scales.
*   **Performance:** Reported to converge quickly and achieve real-time rendering speeds [S1, unofficial]. Training time is reported as 8 minutes for D-NeRF datasets and 30 minutes for HyperNeRF datasets on corrected configurations [S1, unofficial].
*   **Strengths:** Efficient representation for dynamic scenes; real-time rendering capability.
*   **Limitations:** Limited to 2D and 3D grid sampling; requires careful configuration of plane combinations.

## Background
HexPlaneField is part of the 4D Gaussians framework, accepted by CVPR 2024 [S1, unofficial]. It was developed by researchers from HUST and Huawei Inc. [S1, unofficial]. The method was released in early 2024, with code cleaned and parameter explanations added in June 2024 [S1, unofficial].

## Sources
[S1] ElenaViewSynthesis/4dgs, https://github.com/ElenaViewSynthesis/4dgs, fetched 2026-10-28 (unofficial)
