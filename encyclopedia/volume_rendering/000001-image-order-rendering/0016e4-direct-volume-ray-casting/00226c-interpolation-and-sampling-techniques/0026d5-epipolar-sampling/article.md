# Epipolar Sampling

Also called: Epipolar Line Sampling, Cross-View Epipolar Aggregation

## Overview
Epipolar Sampling is a technique to sample features from other views along epipolar lines for a given ray, enabling cross-view feature aggregation without full 3D volume rendering during the encoding phase [S1, unofficial]. It differs from standard volume rendering by operating in 2D image space using epipolar geometry to align features from source views to a target view, rather than integrating density along 3D rays in world space [S1, unofficial].

## How it works
**Inputs:**
- Source view images and their camera parameters (intrinsics $K$, extrinsics $W_{cam}$).
- Target view ray origins and directions (or pixel coordinates to generate them).
- Number of samples per epipolar line segment, $s$ (Typical value: 64 [S1, unofficial]).

**Outputs:**
- Sampled features from other views.
- Validity masks for the samples.

**Data Structures:**
The method produces tensors with the following layout [S1, unofficial]:
- `features`: `Float[Tensor, 'batch view other_view ray sample channel']`
- `valid`: `Bool[Tensor, 'batch view other_view ray']`
- `xy_ray`: `Float[Tensor, 'batch view ray 2']`
- `xy_sample`: `Float[Tensor, 'batch view other_view ray sample 2']`
- `xy_sample_near`: `Float[Tensor, 'batch view other_view ray sample 2']`
- `xy_sample_far`: `Float[Tensor, 'batch view other_view ray sample 2']`
- `origins`: `Float[Tensor, 'batch view ray 3']`
- `directions`: `Float[Tensor, 'batch view ray 3']`

**Steps:**
1. **Generate Image Rays:** For each pixel in the source view, generate world-space ray origins and directions. This is done by sampling an image grid and transforming pixel coordinates to world rays using intrinsic and extrinsic parameters [S1, unofficial].
   - Let $N$ be the number of rays (pixels) and $V$ be the number of views.
   - Compute `origins` and `directions` for all rays in the current view.

2. **Project Rays to Other Views:** Project the generated rays onto all other target views using their camera parameters (extrinsics, intrinsics, near, and far planes) [S1, unofficial].
   - For each target view $v'$, compute the intersection of the ray with the image plane.
   - Determine the bounding box of the projected ray segment in the target view's image coordinates: $xy_{min}$ and $xy_{max}$ [S1, unofficial].
   - Compute an overlap mask `overlaps_image` indicating which rays actually intersect the target view's image bounds [S1, unofficial].

3. **Clamp and Mask:**
   - Clamp $xy_{min}$ and $xy_{max}$ to zero for any infinite values resulting from parallel rays or projection artifacts [S1, unofficial].
   - Apply the `overlaps_image` mask to filter out invalid projections [S1, unofficial].

4. **Compute Sample Depths:** Generate $s$ sample depths linearly spaced between 0 and 1 along the epipolar line segment [S1, unofficial].
   - Formula:
     $$ d_i = \frac{i + 0.5}{s}, \quad i = 0, \dots, s-1 $$
   - In code: `sample_depth = (torch.arange(s) + 0.5) / s` [S1, unofficial].

5. **Interpolate Sample Coordinates:** Interpolate the 2D coordinates in the target view for each sample depth [S1, unofficial].
   - Formula:
     $$ xy_{sample} = xy_{min} + d_i \cdot (xy_{max} - xy_{min}) $$
   - This yields `xy_sample_near` and `xy_sample_far` if near/far clipping is applied, or simply the interpolated points along the line segment [S1, unofficial].

6. **Handle View Exclusion:** Transpose the sample coordinates so the 'view' dimension refers to the target view from which samples are drawn. Exclude the diagonal (self-view) using precomputed indices to avoid sampling from the source view itself [S1, unofficial].

7. **Sample Features:** Sample image features at the computed `xy_sample` coordinates using bilinear interpolation (`grid_sample` in PyTorch) with zero padding for out-of-bounds coordinates [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def epipolar_sampling(origins, directions, intrinsics, extrinsics, num_samples=64):
    """
    origins: [B, V, R, 3] - Ray origins in world space
    directions: [B, V, R, 3] - Ray directions in world space
    intrinsics: [B, V, 3, 3] - Camera intrinsics
    extrinsics: [B, V, 4, 4] - Camera extrinsics (OpenCV-style: camera-to-world)
    num_samples: int - Number of samples per epipolar line
    """
    B, V, R, _ = origins.shape
    device = origins.device
    
    # Step 1: Generate rays (already provided as input in this simplified signature)
    # In practice, these are derived from pixel coordinates and camera params
    
    # Step 2: Project rays to other views
    # We need to project each ray from view v to all other views v'
    # This involves transforming ray origins and directions to the target view's coordinate system
    
    xy_min_list = []
    xy_max_list = []
    overlaps_list = []
    
    for v_prime in range(V):
        if v_prime == v: # Skip self-view for aggregation, but keep structure
            continue
            
        # Get target view camera params
        K_prime = intrinsics[:, v_prime, :, :] # [B, 3, 3]
        W_c2w_prime = extrinsics[:, v_prime, :, :] # [B, 4, 4]
        W_w2c_prime = torch.inverse(W_c2w_prime) # [B, 4, 4]
        
        # Transform ray origins and directions to target view
        # origins: [B, V, R, 3] -> [B, V, R, 4] (homogeneous)
        origins_h = torch.cat([origins, torch.ones_like(origins[..., :1])], dim=-1) # [B, V, R, 4]
        directions_h = torch.cat([directions, torch.zeros_like(directions[..., :1])], dim=-1) # [B, V, R, 4]
        
        origins_w2c = (W_w2c_prime.unsqueeze(1) @ origins_h.unsqueeze(-1)).squeeze(-1) # [B, V, R, 4]
        directions_w2c = (W_w2c_prime.unsqueeze(1) @ directions_h.unsqueeze(-1)).squeeze(-1) # [B, V, R, 4]
        
        origins_t = origins_w2c[..., :3] # [B, V, R, 3]
        directions_t = directions_w2c[..., :3] # [B, V, R, 3]
        
        # Project to 2D image coordinates
        # x = K * (origin + t * direction)
        # We need to find the range of t that intersects the image plane (z=1 in camera coords)
        # t = (1 - origin_z) / direction_z
        
        # Avoid division by zero
        eps = 1e-8
        t_vals = (1.0 - origins_t[..., 2:3]) / (directions_t[..., 2:3] + eps) # [B, V, R, 1]
        
        # 2D coordinates in target view
        xy_proj = (K_prime.unsqueeze(1) @ (origins_t + t_vals * directions_t).unsqueeze(-1)).squeeze(-1) # [B, V, R, 3]
        xy_2d = xy_proj[..., :2] / (xy_proj[..., 2:3] + eps) # [B, V, R, 2]
        
        # Compute min and max xy over the ray segment (if near/far planes are defined, use them)
        # For simplicity, assume full ray or use min/max of projected points
        # In pixelSplat, this is handled by project_rays which returns xy_min, xy_max, overlaps_image
        # Here we approximate:
        xy_min = torch.min(xy_2d, dim=-2, keepdim=True)[0] # [B, V, 1, 2] -> broadcast to [B, V, R, 2]
        xy_max = torch.max(xy_2d, dim=-2, keepdim=True)[0] # [B, V, 1, 2] -> broadcast to [B, V, R, 2]
        
        # Check if ray overlaps image (simplified)
        # Real implementation uses near/far planes and bounding box checks
        overlaps = torch.ones_like(xy_min[..., 0]) # [B, V, R]
        
        xy_min_list.append(xy_min)
        xy_max_list.append(xy_max)
        overlaps_list.append(overlaps)
        
    # Stack results for all target views
    # Note: The actual implementation handles the 'other_view' dimension explicitly
    # This pseudocode is a simplified representation of the vectorized logic
    
    # Step 3: Clamp and Mask
    # Clamp infinite values to 0
    xy_min = torch.where(torch.isinf(xy_min), torch.zeros_like(xy_min), xy_min)
    xy_max = torch.where(torch.isinf(xy_max), torch.zeros_like(xy_max), xy_max)
    
    # Step 4: Compute sample depths
    sample_depth = (torch.arange(num_samples, device=device) + 0.5) / num_samples # [s]
    
    # Step 5: Interpolate sample coordinates
    # xy_sample = xy_min + sample_depth * (xy_max - xy_min)
    # This needs to be broadcasted correctly
    # xy_min: [B, V, R, 2], xy_max: [B, V, R, 2], sample_depth: [s]
    # Result: [B, V, R, s, 2]
    
    # Step 6: Handle view exclusion and transpose
    # Exclude self-view and transpose so 'view' dim refers to target view
    
    # Step 7: Sample features
    # Use grid_sample with zero padding
    # xy_sample needs to be normalized to [-1, 1] for grid_sample
    # features = F.grid_sample(image, xy_sample_normalized, mode='bilinear', padding_mode='zeros')
    
    return features, valid
```

**Edge Cases and Pitfalls:**
- **Infinite Values:** Rays parallel to the image plane result in infinite $t$ values. These must be clamped to zero or masked out [S1, unofficial].
- **Out-of-Bounds:** Samples falling outside the target image bounds must be handled by zero-padding in `grid_sample` [S1, unofficial].
- **Self-View:** The diagonal (sampling from the source view itself) must be excluded to prevent identity mapping artifacts [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(V^2 \cdot R \cdot s)$, where $V$ is the number of views, $R$ is the number of rays (pixels), and $s$ is the number of samples per epipolar line.
- **Memory Complexity:** $O(V^2 \cdot R \cdot s \cdot C)$, where $C$ is the feature channel size.
- **Strengths:** Efficient cross-view feature aggregation without explicit 3D reconstruction. Leverages epipolar geometry for precise alignment.
- **Limitations:** Requires accurate camera parameters. Performance degrades with large baseline differences or inaccurate extrinsics.
- **When to use:** Sparse-view novel view synthesis, multi-view feature encoding.
- **When not to use:** Dense volume rendering with complex lighting, scenes with significant parallax errors.

## Background
Epipolar sampling is a core component of the pixelSplat architecture, introduced by Charatan et al. in 2023 [S1, unofficial]. It enables scalable generalizable 3D reconstruction by aggregating features from multiple views along epipolar lines, avoiding the need for explicit 3D volume rendering during the encoding phase [S1, unofficial]. The method builds on earlier work in multi-view stereo and neural rendering, adapting epipolar constraints for deep learning-based feature aggregation.

## Sources
[S1] dcharatan/pixelsplat, https://github.com/dcharatan/pixelsplat, fetched 2026-10-27 (unofficial)
