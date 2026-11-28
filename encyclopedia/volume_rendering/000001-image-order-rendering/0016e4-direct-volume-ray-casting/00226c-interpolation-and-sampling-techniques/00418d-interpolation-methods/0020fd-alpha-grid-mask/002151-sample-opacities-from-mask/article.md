# Sample Opacities from Mask

Also called: TensorBase Alpha Computation, Volume Alpha

## Overview
Samples opacity values from a mask volume for each point position, mapping raw mask values to a specified opacity range [S1, unofficial]. It builds on [Alpha Grid Mask](2430-alpha-grid-mask.md) by providing the specific interpolation and scaling logic required to convert raw mask data into usable opacity values for volume rendering or culling [G].

## How it works
**Inputs:**
- `positions`: Tensor of shape $[N, 3]$ containing the 3D coordinates of the points to sample [S1, unofficial].
- `alpha_volume`: The 3D alpha mask volume tensor [S2, unofficial].
- `min_opacity`: Minimum opacity value (default 0.0) [S1, unofficial].
- `max_opacity`: Maximum opacity value (default 1.0) [S1, unofficial].

**Outputs:**
- `opacities`: Tensor of shape $[N]$ containing the mapped opacity values for each point [G].

**Data Structures:**
The method relies on the `AlphaGridMask` structure [S2, unofficial]. Key fields include `aabb` (tensor, shape $[2, 3]$), `aabbSize` (tensor, `aabb[1] - aabb[0]`), `invgridSize` (float, `1.0 / aabbSize * 2`), and `alpha_volume` (tensor, viewed as `[1, 1, *shape[-3:]]`) [S2, unofficial].

**Steps:**
1. **Normalization**: Convert `positions` from world space to the normalized grid space of the `alpha_volume` using `aabb` and `invgridSize` [G].
2. **Clamping**: Clamp the normalized coordinates to the range $[0, 1]$ to handle points outside the AABB [G].
3. **Sampling**: Sample the `alpha_volume` at the normalized positions using trilinear interpolation [S3, unofficial].
4. **Mapping**: Map the raw sampled values $r$ to the target opacity range using the linear transformation:
   $$ o = \text{min\_opacity} + r \times (\text{max\_opacity} - \text{min\_opacity}) $$
   where $o$ is the final opacity [S1, unofficial].
5. **Clamping**: Clamp the resulting opacities to $[\text{min\_opacity}, \text{max\_opacity}]$ to ensure numerical stability and validity [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def sample_opacities_from_mask(positions: torch.Tensor, aabb: torch.Tensor, alpha_volume: torch.Tensor, invgridSize: float, min_opacity: float = 0.0, max_opacity: float = 1.0) -> torch.Tensor:
    """
    Samples opacities from a mask volume for given 3D positions.
    
    Args:
        positions: Tensor of shape [N, 3] in world coordinates.
        aabb: Tensor of shape [2, 3] defining the axis-aligned bounding box.
        alpha_volume: Tensor of the mask volume.
        invgridSize: Float scaling factor from the mask structure.
        min_opacity: Minimum opacity value.
        max_opacity: Maximum opacity value.
        
    Returns:
        Tensor of shape [N] with sampled opacity values.
    """
    # 1 & 2. Normalize to [0, 1] range based on AABB and clamp
    aabb_min = aabb[0]  # [3]
    aabb_max = aabb[1]  # [3]
    pos_norm = (positions - aabb_min) / (aabb_max - aabb_min)
    pos_norm = torch.clamp(pos_norm, 0.0, 1.0)

    # 3. Sample alpha_volume using trilinear interpolation
    # grid_sample expects grid of shape [N, H_out, W_out, D_out, 3]
    # We sample at single points per position, so H_out=W_out=D_out=1
    grid = pos_norm.unsqueeze(0).unsqueeze(1).unsqueeze(2)  # [1, 1, 1, 1, 3]
    sampled = torch.nn.functional.grid_sample(
        alpha_volume, grid, mode='trilinear', align_corners=True
    )
    raw_opacities = sampled.squeeze()  # [N]

    # 4. Map raw values to opacity range
    opacities = min_opacity + raw_opacities * (max_opacity - min_opacity)

    # 5. Clamp to valid range
    opacities = torch.clamp(opacities, min=min_opacity, max=max_opacity)

    return opacities
```

**Parameters:**
- `min_opacity`: Minimum opacity value to ensure visibility. Default: 0.0 [S1, unofficial].
- `max_opacity`: Maximum opacity value to prevent complete occlusion. Default: 1.0 [S1, unofficial].

**Edge Cases:**
- Points outside the AABB: The normalization step clamps coordinates to $[0, 1]$ to avoid sampling undefined regions [G].
- Degenerate AABB: If `aabbSize` is zero, `invgridSize` becomes undefined; a guard should check for this [G].

## Subtopics

### Update Opacities from Mask
Updates opacity values for points based on their current positions using a mask volume [S1, unofficial]. It ensures points are in $[N, 3]$ format, computes `mask_min` and `mask_max` from the mask tensor, and then calls `sample_opacities_from_mask` to get the new opacities [S1, unofficial]. This variant is useful for dynamic scenes where point positions change and opacities need to be refreshed relative to the mask [G].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] ValentinKraft/vgs, https://github.com/ValentinKraft/vgs, fetched 2026-10-27 (unofficial)
[S2] GitHub - apchenstu/TensoRF: [ECCV 2022] Tensorial Radiance Fields, a ..., https://github.com/apchenstu/TensoRF, fetched 2026-10-29 (unofficial)
[S3] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
