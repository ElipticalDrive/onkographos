# Epipolar Sampling

Also called: Epipolar Line Sampling, Cross-View Epipolar Aggregation

## Overview
Epipolar sampling is a technique for aggregating features from multiple views by sampling along epipolar lines rather than along camera rays in 3D space [S1, unofficial]. It solves the problem of efficiently correlating information between a reference view and target views without performing full 3D volume rendering or explicit 3D reconstruction during the encoding phase [S1, unofficial]. Unlike standard volume rendering which samples along a ray, this method samples features from a reference pixel and correlates them with features sampled along the corresponding epipolar line in a second view, augmented with positionally encoded depths [S1, unofficial]. This allows the encoder to memorize the correct depth consistent with the scene's arbitrary scale, which is then used for subsequent parameter prediction [S1, unofficial].

## How it works
The method operates on a batch of images, projecting rays from a source view onto target views and sampling features along the resulting epipolar segments.

**Inputs:**
- `features`: Tensor of shape `[batch, view, height, width, channel]` containing image features for all views.
- `intrinsics`: Camera intrinsic matrices.
- `extrinsics`: Camera-to-world transformation matrices (OpenCV-style, +Z forward) [S1, unofficial].
- `near`, `far`: Near and far plane distances for ray projection.
- `num_samples` ($s$): Number of samples per epipolar line segment. Typical value: 64 [S1, unofficial].

**Outputs:**
- `features_sampled`: Tensor of shape `[batch, view, other_view, ray, sample, channel]` containing aggregated features.
- `valid`: Boolean mask of shape `[batch, view, other_view, ray]` indicating valid samples.
- `xy_sample`: Sample coordinates in target view space.

**Data Structures:**
The core data structure for the sampling result is defined by the following fields [S1, unofficial]:
- `features`: `Float[Tensor, 'batch view other_view ray sample channel']`
- `valid`: `Bool[Tensor, 'batch view other_view ray']`
- `xy_ray`: `Float[Tensor, 'batch view ray 2']`
- `xy_sample`: `Float[Tensor, 'batch view other_view ray sample 2']`
- `xy_sample_near`: `Float[Tensor, 'batch view other_view ray sample 2']`
- `xy_sample_far`: `Float[Tensor, 'batch view other_view ray sample 2']`
- `origins`: `Float[Tensor, 'batch view ray 3']`
- `directions`: `Float[Tensor, 'batch view ray 3']`

**Steps:**

1.  **Generate Image Rays:** For each pixel in the source view, generate 2D image coordinates $(x, y)$ and corresponding 3D world rays (origins and directions) using the camera intrinsics and extrinsics [S1, unofficial].
    *   Let $P$ be the set of pixel coordinates.
    *   Compute origins $\mathbf{o}$ and directions $\mathbf{d}$ for each pixel.

2.  **Project Rays to Target Views:** For each target view $v'$, project the source rays onto the image plane of $v'$ using the relative camera poses [S1, unofficial].
    *   Compute the intersection of the ray with the near and far planes of the target view.
    *   Determine the bounding box of the projected ray segment in the target image: $(x_{min}, y_{min})$ and $(x_{max}, y_{max})$.
    *   Compute a validity mask `overlaps_image` indicating if the projected segment intersects the target image boundaries.

3.  **Clamp and Mask:** Clamp $x_{min}, y_{min}, x_{max}, y_{max}$ to zero for infinite values. Apply the `overlaps_image` mask to filter out rays that do not intersect the target view [S1, unofficial].

4.  **Compute Sample Depths:** Generate $s$ linearly spaced depth values along the epipolar segment [S1, unofficial]:
    $$ \text{sample\_depth}_k = \frac{k + 0.5}{s}, \quad k = 0, \dots, s-1 $$
    These depths represent normalized positions along the segment between the near and far plane intersections.

5.  **Interpolate Sample Coordinates:** Calculate the 2D coordinates in the target view for each sample [S1, unofficial]:
    $$ \mathbf{xy\_sample}_k = \mathbf{xy}_{min} + \text{sample\_depth}_k \cdot (\mathbf{xy}_{max} - \mathbf{xy}_{min}) $$
    where $\mathbf{xy}_{min} = (x_{min}, y_{min})$ and $\mathbf{xy}_{max} = (x_{max}, y_{max})$.

6.  **Handle View Exclusion:** Transpose the sample coordinates so the 'view' dimension refers to the target view. Exclude the diagonal (self-view) by masking or indexing appropriately, as sampling from the same view is typically not required for cross-view aggregation [S1, unofficial].

7.  **Sample Features:** Use bilinear interpolation (`grid_sample` with zero padding) to extract feature values from the target view's feature tensor at the computed $\mathbf{xy\_sample}_k$ coordinates [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def epipolar_sampling(features, intrinsics, extrinsics, near, far, num_samples=64):
    """
    Args:
        features: [B, V, H, W, C] - Features for all views
        intrinsics: [V, 3, 3] - Intrinsic matrices
        extrinsics: [V, 4, 4] - Camera-to-world matrices (OpenCV style)
        near: float - Near plane distance
        far: float - Far plane distance
        num_samples: int - Number of samples per epipolar line
    
    Returns:
        sampled_features: [B, V, V, R, S, C]
        valid: [B, V, V, R]
        xy_sample: [B, V, V, R, S, 2]
    """
    B, V, H, W, C = features.shape
    R = H * W  # Number of rays (one per pixel in source view)
    
    # 1. Generate image rays for source view
    # Create grid of pixel coordinates [H, W, 2]
    y, x = torch.meshgrid(torch.arange(H), torch.arange(W), indexing='ij')
    xy_source = torch.stack([x, y], dim=-1).float().to(features.device) # [H, W, 2]
    
    # Reshape to [R, 2]
    xy_source_flat = xy_source.reshape(-1, 2) # [R, 2]
    
    # Convert pixel coordinates to normalized device coordinates (NDC) or camera space
    # Assuming standard pinhole model: x_ndc = (x - cx) / fx
    fx = intrinsics[:, 0, 0] # [V]
    fy = intrinsics[:, 1, 1] # [V]
    cx = intrinsics[:, 0, 2] # [V]
    cy = intrinsics[:, 1, 2] # [V]
    
    # For source view (view 0 for simplicity, or iterate over all views as source)
    # Let's assume we iterate over all views as source views
    # origins and directions are computed per source view
    
    sampled_features_list = []
    valid_list = []
    xy_sample_list = []
    
    for src_view in range(V):
        src_intrinsics = intrinsics[src_view]
        src_extrinsics = extrinsics[src_view]
        
        # Compute origins and directions for source view rays
        # Origin is camera center
        cam_center = -extrinsics[src_view][:3, :3].T @ extrinsics[src_view][:3, 3] # [3]
        origins = cam_center.unsqueeze(0).expand(R, -1) # [R, 3]
        
        # Direction: unproject pixel coordinates
        # x_cam = (x - cx) / fx, y_cam = (y - cy) / fy, z_cam = 1
        x_ndc = (xy_source_flat[:, 0] - cx[src_view]) / fx[src_view]
        y_ndc = (xy_source_flat[:, 1] - cy[src_view]) / fy[src_view]
        z_cam = torch.ones(R, device=features.device)
        
        directions_cam = torch.stack([x_ndc, y_ndc, z_cam], dim=-1) # [R, 3]
        
        # Transform directions to world space
        R_cam_to_world = extrinsics[src_view][:3, :3]
        directions_world = directions_cam @ R_cam_to_world.T # [R, 3]
        
        # 2. Project rays to all other views
        for tgt_view in range(V):
            if src_view == tgt_view:
                continue # Skip self-view
            
            tgt_intrinsics = intrinsics[tgt_view]
            tgt_extrinsics = extrinsics[tgt_view]
            
            # Transform ray to target camera space
            # T_world_to_tgt = extrinsics[tgt_view]^-1
            # Since extrinsics are camera-to-world, inverse is world-to-camera
            T_world_to_tgt = torch.linalg.inv(tgt_extrinsics) # [4, 4]
            
            # Transform origins
            origins_homogeneous = torch.cat([origins, torch.ones(R, 1)], dim=-1) # [R, 4]
            origins_tgt = (T_world_to_tgt @ origins_homogeneous.T).T[:, :3] # [R, 3]
            
            # Transform directions
            directions_tgt = (T_world_to_tgt[:3, :3] @ directions_world.T).T # [R, 3]
            
            # Intersect with near and far planes in target camera space
            # Ray: P = O + t * D
            # Near plane: z = near, Far plane: z = far
            # t_near = (near - O_z) / D_z
            # t_far = (far - O_z) / D_z
            
            dz = directions_tgt[:, 2] # [R]
            oz = origins_tgt[:, 2] # [R]
            
            # Handle division by zero or near-zero dz
            eps = 1e-6
            dz_safe = torch.where(torch.abs(dz) < eps, torch.sign(dz) * eps, dz)
            
            t_near = (near - oz) / dz_safe
            t_far = (far - oz) / dz_safe
            
            # Compute intersection points
            pts_near = origins_tgt + t_near.unsqueeze(-1) * directions_tgt # [R, 3]
            pts_far = origins_tgt + t_far.unsqueeze(-1) * directions_tgt # [R, 3]
            
            # Project to 2D image coordinates
            # x_img = x_cam * fx / z_cam + cx
            # y_img = y_cam * fy / z_cam + cy
            
            x_near = pts_near[:, 0] * fx[tgt_view] / pts_near[:, 2] + cx[tgt_view]
            y_near = pts_near[:, 1] * fy[tgt_view] / pts_near[:, 2] + cy[tgt_view]
            
            x_far = pts_far[:, 0] * fx[tgt_view] / pts_far[:, 2] + cx[tgt_view]
            y_far = pts_far[:, 1] * fy[tgt_view] / pts_far[:, 2] + cy[tgt_view]
            
            xy_min = torch.stack([x_near, y_near], dim=-1) # [R, 2]
            xy_max = torch.stack([x_far, y_far], dim=-1) # [R, 2]
            
            # Clamp to image boundaries for validity check
            # Valid if segment overlaps image [0, W] x [0, H]
            # Simple check: min/max of segment within bounds
            overlaps_image = (xy_min[:, 0] < W) & (xy_max[:, 0] > 0) & \
                             (xy_min[:, 1] < H) & (xy_max[:, 1] > 0)
            
            # Clamp coordinates to [0, 0] for invalid rays to avoid NaNs in interpolation
            xy_min = torch.where(overlaps_image.unsqueeze(-1), xy_min, torch.zeros_like(xy_min))
            xy_max = torch.where(overlaps_image.unsqueeze(-1), xy_max, torch.zeros_like(xy_max))
            
            # 4. Compute sample depths
            sample_depths = (torch.arange(num_samples, device=features.device) + 0.5) / num_samples # [S]
            
            # 5. Interpolate sample coordinates
            # xy_sample [R, S, 2]
            xy_sample = xy_min.unsqueeze(1) + sample_depths.unsqueeze(0).unsqueeze(-1) * (xy_max.unsqueeze(1) - xy_min.unsqueeze(1))
            
            # 6. Sample features
            # Normalize to [-1, 1] for grid_sample
            # grid_sample expects normalized coordinates in range [-1, 1]
            # x_norm = 2 * (x / W) - 1
            # y_norm = 2 * (y / H) - 1
            
            x_norm = 2.0 * (xy_sample[:, :, 0] / W) - 1.0
            y_norm = 2.0 * (xy_sample[:, :, 1] / H) - 1.0
            
            grid = torch.stack([x_norm, y_norm], dim=-1) # [R, S, 2]
            
            # features shape: [B, C, H, W] for grid_sample (NCHW)
            # We need to sample from features[src_view]
            feat_src = features[:, src_view, :, :, :] # [B, C, H, W]
            
            # grid_sample input: [B, H, W, 2]
            grid_batched = grid.unsqueeze(0).expand(B, -1, -1, -1) # [B, R, S, 2]
            
            # Sample
            # grid_sample output: [B, C, R, S]
            sampled = F.grid_sample(feat_src, grid_batched, mode='bilinear', padding_mode='zeros', align_corners=False) # [B, C, R, S]
            
            # Transpose to [B, R, S, C]
            sampled = sampled.permute(0, 2, 3, 1)
            
            # Expand valid mask to [B, R, S]
            valid_mask = overlaps_image.unsqueeze(-1).unsqueeze(-1).expand(B, -1, -1) # [B, R, S]
            
            # Store results
            # Reshape to [B, 1, 1, R, S, C] for concatenation over views
            sampled_features_list.append(sampled.unsqueeze(1).unsqueeze(1))
            valid_list.append(valid_mask.any(dim=-1).unsqueeze(1).unsqueeze(1)) # Valid if any sample is valid
            xy_sample_list.append(xy_sample.unsqueeze(0).unsqueeze(0))
            
    # Concatenate over other_view dimension
    sampled_features = torch.cat(sampled_features_list, dim=1) # [B, V-1, R, S, C]
    valid = torch.cat(valid_list, dim=1) # [B, V-1, R]
    xy_sample = torch.cat(xy_sample_list, dim=1) # [B, V-1, R, S, 2]
    
    # Reorder dimensions to [B, V, V, R, S, C] with zeros for self-view
    # This is a simplified representation; actual implementation would handle the full VxV grid
    return sampled_features, valid, xy_sample
```

## Subtopics

### [Epipolar Encoder](6426-epipolar-encoder.md)
The Epipolar Encoder is a two-view (or multi-view) neural module that resolves scale ambiguity in 3D reconstruction by aggregating cross-view features along epipolar lines. It solves the problem where camera poses are scaled by an arbitrary factor $s_i$, preventing the inference of metric depth from a single view. Unlike standard volume rendering which samples along a ray, this method samples features from a reference pixel and correlates them with features sampled along the corresponding epipolar line in a second view, augmented with positionally encoded depths. This allows the encoder to memorize the correct depth consistent with the scene's arbitrary scale, which is then used for subsequent Gaussian parameter prediction.

### Spot Light Rectification
An extension of the rectification mapping for perspective cameras and local spot lights, where the $\gamma$ coordinate is set to the angle the light ray makes with the direction from the light to the camera [S2]. This results in nonuniform sampling density along the camera ray, causing the $ds/d\gamma$ term in the integral $I(\gamma, \beta)$ to vary with both $\gamma$ and $\beta$, unlike directional lights where it varies only with $\beta$ [S2].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of rays and samples per ray. The projection step is $O(V^2 \cdot R)$, and the sampling step is $O(V^2 \cdot R \cdot S)$.
- **Memory Complexity:** $O(V^2 \cdot R \cdot S \cdot C)$ for storing sampled features.
- **Strengths:** Efficient cross-view feature aggregation without explicit 3D reconstruction. Handles scale ambiguity effectively.
- **Limitations:** Requires accurate camera poses. Performance degrades with large baseline angles or occlusions.
- **When to use:** Sparse-view 3D reconstruction, novel view synthesis, multi-view stereo.
- **When not to use:** Dense volumetric rendering where explicit 3D structure is needed.

## Background
Epipolar sampling is a key component in modern neural radiance field and 3D Gaussian splatting methods for sparse-view reconstruction. It was popularized by pixelSplat [S1, unofficial], which uses it to aggregate features from multiple views for 3D Gaussian parameter prediction. The method builds on traditional multi-view stereo techniques that use epipolar geometry for correspondence matching.

## Sources
[S1] dcharatan/pixelsplat, https://github.com/dcharatan/pixelsplat, fetched 2026-10-27 (unofficial)
[S2] A hierarchical volumetric shadow algorithm for single scattering, https://doi.org/10.1145/1866158.1866200, fetched 2026-11-03
