# Alpha Computation with Alpha Mask

Also called: Masked Polygons and $\bar{\alpha}$, $\bar{\alpha}$ Occlusion Weighing

## Overview
This method computes the alpha value for volume rendering or occlusion estimation by applying a binary mask derived from an alpha texture, ensuring that only valid surface regions contribute to the density or opacity accumulation. It solves the problem of over-estimating occlusion or density for planar surfaces with complex contours (e.g., foliage, fences) where the underlying geometry does not match the actual occlusion properties of the surface [S2, unofficial]. Unlike siblings that convert raw density directly to alpha or extinction, this method gates the computation using a pre-defined alpha mask, preventing invalid background areas from contributing to the final result [S1, unofficial].

## How it works
**Inputs:**
- A set of 3D locations `xyz_locs` (tensor of shape $[N, 3]$).
- An alpha mask texture (or tensor) associated with the surface.
- Density features or raw density values at `xyz_locs`.
- Integration length `length` (scalar or tensor of shape $[N]$).
- Boolean flag `component_wise_feature2density` indicating if features are already densities.

**Outputs:**
- A sigma tensor (absorption coefficient) of shape $[N]$ with zeros in masked-out regions.
- An alpha tensor of shape $[N]$ computed from the valid sigma values.

**Data Structures:**
- `alpha_mask`: Boolean tensor of shape $[N]$, where `True` indicates the location is within the valid alpha region.
- `sigma`: Float tensor of shape $[N]$, initialized to zeros.

**Steps:**
1. **Mask Generation:** If an alpha mask exists, sample the alpha values at `xyz_locs`. Create a boolean mask `alpha_mask` where `alpha > 0`. [S1, unofficial]
2. **Initialization:** Initialize the `sigma` tensor with zeros. [S1, unofficial]
3. **Valid Point Check:** Determine if there are any valid points using `alpha_mask.any()`. If no valid points exist, return zeros. [S1, unofficial]
4. **Feature Processing:** For points where `alpha_mask` is true:
    - Normalize coordinates if required.
    - Compute density features.
    - If `component_wise_feature2density` is `True`, use the features directly as `sigma_feature`. Otherwise, apply the `feature2density` conversion function. [S1, unofficial]
5. **Sigma Assignment:** Assign the computed `sigma_feature` values to the `sigma` tensor at indices where `alpha_mask` is true. Indices where `alpha_mask` is false remain zero. [S1, unofficial]
6. **Alpha Computation:** Compute the final alpha for all points using the Beer-Lambert law:
   $$ \alpha = 1 - \exp(-\sigma \cdot \text{length}) $$
   [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_alpha_with_mask(xyz_locs, alpha_texture, features, length, component_wise_feature2density, feature2density_func):
    # Step 1: Generate mask
    # Sample alpha values at xyz_locs from alpha_texture
    alpha_values = sample_texture(alpha_texture, xyz_locs)
    alpha_mask = alpha_values > 0.0
    
    # Step 2: Initialize sigma
    sigma = torch.zeros_like(features[..., 0]) # Assuming features[..., 0] is the density component
    
    # Step 3: Check for valid points
    if not alpha_mask.any():
        return torch.zeros_like(alpha_mask, dtype=torch.float32)
    
    # Step 4 & 5: Compute and assign sigma for valid points
    valid_indices = torch.where(alpha_mask)[0]
    
    if len(valid_indices) > 0:
        # Get features for valid points
        valid_features = features[valid_indices]
        
        # Convert to sigma
        if component_wise_feature2density:
            valid_sigma = valid_features[..., 0] # Assuming first component is density
        else:
            valid_sigma = feature2density_func(valid_features)
            
        # Assign to sigma tensor
        sigma[valid_indices] = valid_sigma
        
    # Step 6: Compute alpha
    # Handle potential division by zero or invalid length if necessary, 
    # though length is typically positive.
    alpha = 1.0 - torch.exp(-sigma * length)
    
    return alpha
```

**Parameters:**
- `component_wise_feature2density`: Boolean. If `True`, skips density conversion. [S1, unofficial]
- `feature2density_func`: Function to convert raw features to density. [S1, unofficial]

**Edge Cases and Pitfalls:**
- If `alpha_mask` is all false, the result is all zeros. [S1, unofficial]
- Precision issues may arise if `sigma * length` is very large, causing `exp` to underflow to zero, resulting in alpha 1.0. [G]
- The method assumes the alpha mask accurately represents the valid surface area. [S2, unofficial]

## Subtopics

### [Alpha Discard](14786-alpha-discard.md)
Alpha Discard is a fragment-level optimization in direct volume ray casting and voxel rendering that discards fragments with alpha values below a specified threshold. It solves the problem of rendering transparency efficiently by allowing the renderer to skip expensive blending operations for nearly transparent pixels, thereby enabling the rendering engine to pass through to opaque geometry behind them. It is distinct from Alpha Blend, which computes actual transparency with proper accumulation but suffers from sorting and performance limitations when multiple transparent surfaces are involved.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of sample points, dominated by texture sampling and the exponential function.
- **Memory Complexity:** $O(N)$ for storing the `sigma` and `alpha` tensors.
- **Strengths:** Efficiently handles complex geometries like foliage by preventing over-estimation of occlusion. [S2, unofficial]
- **Limitations:** Requires pre-computed or accessible alpha mask. Cannot efficiently handle positional variation within a single polygon without tessellation, as the analytic solution assumes constant opacity over the integration domain. [S2, unofficial]
- **When to use:** When rendering masked polygons (e.g., fences, foliage) where the geometry does not match the occlusion properties. [S2, unofficial]
- **When not to use:** When high-frequency positional variation in opacity is required within a single polygon without tessellation. [S2, unofficial]

## Background
The concept of using an average alpha value $\bar{\alpha}$ to weigh occlusion for masked polygons was introduced in the context of Ambient Occlusion Volumes (AOV) to correct over-estimation of occlusion for surfaces like foliage and fences [S2, unofficial]. The implementation details involving tensor-based computation and feature conversion are found in unofficial repositories such as Joint-TensoRF [S1, unofficial].

## Sources
[S1] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
[S2] Ambient Occlusion Volumes, https://research.nvidia.com/sites/default/files/pubs/2010-06_Ambient-Occlusion-Volumes/McGuire10AOV.pdf, fetched 2026-10-28 (unofficial)
