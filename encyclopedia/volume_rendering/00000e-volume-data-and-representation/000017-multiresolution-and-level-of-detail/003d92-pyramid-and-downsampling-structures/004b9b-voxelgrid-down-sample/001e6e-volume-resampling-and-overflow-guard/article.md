# Volume Resampling and Overflow Guard

Also called: Resampling with Max Voxel Cap, Safe Volume Resampling

## Overview
This method resamples a candidate volume grid to a target shape while enforcing a maximum voxel count to prevent downstream memory overflow [S1, unofficial]. It differs from sibling methods by combining a strict safety cap with precise trilinear interpolation and a fallback dimension-decrement mechanism to guarantee reduction even when rounding constraints interfere [S1, unofficial].

## How it works
**Inputs:**
- Candidate volume grid with dimensions $D_0, H_0, W_0$ [S1, unofficial].
- Candidate voxel count $N_{cand} = D_0 \times H_0 \times W_0$ [S1, unofficial].
- Maximum allowed voxel count $N_{max}$ (default $2^{24} - 1$) [S1, unofficial].
- Optional physical spacing $s_{old} = (s_d, s_h, s_w)$ [S1, unofficial].

**Outputs:**
- Resampled volume grid with adjusted dimensions $D_{adj}, H_{adj}, W_{adj}$ [S1, unofficial].
- Updated physical spacing $s_{new}$ (if input spacing was provided) [S1, unofficial].

**Data Structures:**
- The volume is treated as a tensor of shape $(1, 1, D, H, W)$ during interpolation to support batch and channel dimensions required by standard interpolation libraries [S1, unofficial].

**Steps:**
1. **Check Limit:** If $N_{cand} \le N_{max}$, no resampling is needed; return original dimensions and spacing [S1, unofficial].
2. **Calculate Scale:** Compute the global scaling factor:
   $$ scale = \left( \frac{N_{max}}{N_{cand}} \right)^{1/3} $$ [S1, unofficial].
3. **Compute Adjusted Dimensions:** Calculate initial adjusted dimensions:
   $$ D_{init} = \lfloor D_0 \cdot scale \rfloor $$
   $$ H_{init} = \lfloor H_0 \cdot scale \rfloor $$
   $$ W_{init} = \lfloor W_0 \cdot scale \rfloor $$ [S1, unofficial].
   Apply the minimum dimension constraint:
   $$ D_{adj} = \max(32, D_{init}) $$
   $$ H_{adj} = \max(32, H_{init}) $$
   $$ W_{adj} = \max(32, W_{init}) $$ [S1, unofficial].
4. **Fallback Check:** Calculate the new voxel count $N_{new} = D_{adj} \times H_{adj} \times W_{adj}$ [S1, unofficial]. If $N_{new} \ge N_{cand}$ (i.e., rounding or minimum constraints prevented reduction), decrement each dimension by 1 (minimum 32):
   $$ D_{adj} = \max(32, D_0 - 1) $$
   $$ H_{adj} = \max(32, H_0 - 1) $$
   $$ W_{adj} = \max(32, W_0 - 1) $$ [S1, unofficial].
5. **Resample Volume:** Resample the input volume to shape $(1, 1, D_{adj}, H_{adj}, W_{adj})$ using trilinear interpolation with `align_corners=True` [S1, unofficial]. The volume is unsqueezed to add batch and channel dimensions before interpolation and squeezed back after [S1, unofficial].
6. **Update Spacing:** If physical spacing $s_{old}$ is known, calculate scale factors for each axis to avoid division by zero for size 1 dimensions:
   $$ scale_d = \frac{\max(D_0 - 1, 1)}{\max(D_{adj} - 1, 1)} $$
   $$ scale_h = \frac{\max(H_0 - 1, 1)}{\max(H_{adj} - 1, 1)} $$
   $$ scale_w = \frac{\max(W_0 - 1, 1)}{\max(W_{adj} - 1, 1)} $$ [S1, unofficial].
   Compute new spacing:
   $$ s_{new} = (s_d \cdot scale_d, s_h \cdot scale_h, s_w \cdot scale_w) $$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def resample_volume_with_overflow_guard(volume, dims, max_voxels=2**24 - 1, spacing=None):
    """
    Resample volume to prevent overflow.
    
    Args:
        volume: Tensor of shape (D, H, W) or (1, 1, D, H, W).
        dims: Tuple (D0, H0, W0).
        max_voxels: Maximum allowed voxel count.
        spacing: Tuple (sd, sh, sw) or None.
        
    Returns:
        resampled_volume: Tensor of shape (D_adj, H_adj, W_adj).
        new_dims: Tuple (D_adj, H_adj, W_adj).
        new_spacing: Tuple (sd, sh, sw) or None.
    """
    D0, H0, W0 = dims
    N_cand = D0 * H0 * W0
    
    # Step 1: Check limit
    if N_cand <= max_voxels:
        return volume, dims, spacing
    
    # Step 2: Calculate scale
    scale = (max_voxels / N_cand) ** (1/3)
    
    # Step 3: Compute adjusted dimensions
    D_adj = max(32, math.floor(D0 * scale))
    H_adj = max(32, math.floor(H0 * scale))
    W_adj = max(32, math.floor(W0 * scale))
    
    # Step 4: Fallback check
    N_new = D_adj * H_adj * W_adj
    if N_new >= N_cand:
        D_adj = max(32, D0 - 1)
        H_adj = max(32, H0 - 1)
        W_adj = max(32, W0 - 1)
        
    # Step 5: Resample
    # Ensure volume has batch and channel dims: (1, 1, D, H, W)
    if volume.ndim == 3:
        vol_input = volume.unsqueeze(0).unsqueeze(0)
    else:
        vol_input = volume
        
    # Perform trilinear interpolation with align_corners=True
    # Standard library call: torch.nn.functional.interpolate(vol_input, size=(D_adj, H_adj, W_adj), mode='trilinear', align_corners=True)
    resampled_vol = interpolate_3d_trilinear(vol_input, (D_adj, H_adj, W_adj), align_corners=True)
    
    # Squeeze back to (D, H, W)
    resampled_vol = resampled_vol.squeeze(0).squeeze(0)
    
    # Step 6: Update spacing
    new_spacing = None
    if spacing is not None:
        sd, sh, sw = spacing
        # Avoid division by zero for size 1 dimensions
        scale_d = max(D0 - 1, 1) / max(D_adj - 1, 1)
        scale_h = max(H0 - 1, 1) / max(H_adj - 1, 1)
        scale_w = max(W0 - 1, 1) / max(W_adj - 1, 1)
        
        new_spacing = (sd * scale_d, sh * scale_h, sw * scale_w)
        
    return resampled_vol, (D_adj, H_adj, W_adj), new_spacing

def interpolate_3d_trilinear(tensor, new_size, align_corners=True):
    """
    Standard trilinear interpolation for 3D tensors.
    Resamples the last three dimensions of `tensor` to `new_size`.
    """
    # Implementation delegated to standard tensor library (e.g., torch.nn.functional.interpolate)
    # Returns tensor resampled to new_size along the last 3 dimensions
    pass
```

**Parameters:**
- `max_voxels`: Conservative safety cap for voxel count. Default value is $2^{24} - 1$ (approximately 16 million voxels) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Rounding Prevention:** If the calculated adjusted dimensions result in a voxel count not strictly less than the candidate count (due to rounding down or minimum dimension constraints), the fallback mechanism decrements dimensions by 1 [S1, unofficial].
- **Minimum Dimension:** Dimensions are clamped to a minimum of 32 to prevent degenerate grids [S1, unofficial].
- **Spacing Division by Zero:** When calculating new spacing, `max(dim - 1, 1)` is used to handle cases where a dimension is 1, avoiding division by zero [S1, unofficial].

## Subtopics

## Operating characteristics
- **Time Complexity:** Dominated by the trilinear interpolation step, which is $O(D_{adj} \cdot H_{adj} \cdot W_{adj})$ [G].
- **Memory Complexity:** Requires temporary memory for the unsqueezed volume and the resampled volume [G].
- **Strengths:** Prevents memory overflow in downstream processes; ensures a strict reduction in voxel count [S1, unofficial].
- **Limitations:** Trilinear interpolation may blur high-frequency details; the fallback mechanism may result in suboptimal dimensions if the initial scale calculation is close to integer boundaries [G].
- **When to Use:** When processing large volumes that might exceed memory limits in subsequent steps [S1, unofficial].

## Background
This method is documented in the VGS repository [S1, unofficial]. It addresses the issue of memory overflow when dealing with high-resolution voxel grids [S1, unofficial].

## Sources
[S1] ValentinKraft/vgs, https://github.com/ValentinKraft/vgs, fetched 2026-10-27 (unofficial)
