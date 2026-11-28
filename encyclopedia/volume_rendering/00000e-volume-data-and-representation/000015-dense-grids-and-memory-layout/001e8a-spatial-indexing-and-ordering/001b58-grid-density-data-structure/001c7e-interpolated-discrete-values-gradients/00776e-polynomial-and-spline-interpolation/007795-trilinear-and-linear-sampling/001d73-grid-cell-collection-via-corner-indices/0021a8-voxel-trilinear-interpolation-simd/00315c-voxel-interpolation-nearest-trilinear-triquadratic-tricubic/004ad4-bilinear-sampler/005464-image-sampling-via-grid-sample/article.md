# Image Sampling via Grid Sample


## Overview
This method samples image colors at specific 2D pixel coordinates using PyTorch's `grid_sample` function with bilinear interpolation [S1, unofficial]. It serves as the bridge between ray-casting logic (which produces 2D coordinates in normalized space) and the final pixel color values. It differs from the parent [Bilinear Sampler](12826-bilinear-sampler.md) by enforcing a specific coordinate transformation pipeline (negation and permutation) required by the assignment's ray bundle structure, rather than exposing the full general-purpose API [S1, unofficial].

## How it works
**Inputs:**
- `images`: A tensor of shape `(batch_size, height, width, channels)`.
- `xy_grid`: A tensor of shape `(batch_size, num_samples, 2)` containing normalized 2D coordinates $(x, y)$ for each sample point.

**Outputs:**
- `sampled_colors`: A tensor of shape `(batch_size * num_samples, channels)`.

**Data Structures:**
- `xy_grid`: Normalized coordinates in the range $[-1, 1]$. The source notes imply a specific convention where the grid must be negated before sampling [S1, unofficial].

**Steps:**
1. Extract `batch_size` and `spatial_size` (height, width) from the `images` tensor.
2. Reshape `xy_grid` to `(batch_size, -1, 1, 2)` to match the expected input format for `grid_sample` (batch, grid_height, grid_width, 2).
3. Negate the `xy_grid` coordinates. This step is critical for aligning the coordinate system with the renderer's ray origin/direction logic [S1, unofficial].
4. Permute `images` from `(N, H, W, C)` to `(N, C, H, W)` to match PyTorch's channel-first expectation for `grid_sample`.
5. Call `torch.nn.functional.grid_sample` with:
   - `mode='bilinear'`
   - `padding_mode='zeros'` (implied by standard usage, though not explicitly stated, it is standard for this context)
   - `align_corners=True` (explicitly stated in notes)
6. Permute the result from `(N, C, 1, M)` back to `(N, C, M, 1)` or similar intermediate shape, then view/reshape to `(-1, images.shape[-1])` to flatten the batch and spatial dimensions into a single list of sampled colors [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def image_sampling_via_grid_sample(images: torch.Tensor, xy_grid: torch.Tensor) -> torch.Tensor:
    """
    Samples image colors at specific 2D pixel coordinates using PyTorch's bilinear grid sampling.
    
    Args:
        images: Tensor of shape (batch_size, height, width, channels)
        xy_grid: Tensor of shape (batch_size, num_samples, 2) containing normalized 2D coordinates
        
    Returns:
        sampled_colors: Tensor of shape (batch_size * num_samples, channels)
    """
    # 1. Extract batch_size and spatial_size
    batch_size = images.shape[0]
    # spatial_size is implicitly height and width from images.shape[1] and images.shape[2]
    
    # 2. Reshape xy_grid to (batch_size, -1, 1, 2)
    # -1 infers num_samples, 1 is for grid_height (since we are sampling points, not a grid)
    xy_grid_reshaped = xy_grid.view(batch_size, -1, 1, 2)
    
    # 3. Negate the grid coordinates
    xy_grid_negated = -xy_grid_reshaped
    
    # 4. Permute images to (N, C, H, W) format
    images_permuted = images.permute(0, 3, 1, 2)
    
    # 5. Call torch.nn.functional.grid_sample
    # align_corners=True is specified in the notes
    # padding_mode is not explicitly specified in notes, but 'zeros' is standard for this assignment context
    sampled = F.grid_sample(
        images_permuted, 
        xy_grid_negated, 
        mode='bilinear', 
        padding_mode='zeros', 
        align_corners=True
    )
    
    # 6. Permute result back to (0, 2, 3, 1) and view as (-1, images.shape[-1])
    # sampled shape is (N, C, 1, M) where M is num_samples
    # Permute to (N, C, M, 1) then reshape
    sampled_permuted = sampled.permute(0, 1, 3, 2) # (N, C, M, 1)
    sampled_colors = sampled_permuted.view(-1, images.shape[-1])
    
    return sampled_colors
```

**Edge Cases and Pitfalls:**
- **Coordinate Convention:** The negation step (`-xy_grid`) is non-standard for generic bilinear sampling and is specific to this assignment's ray coordinate system. Failing to negate will result in incorrect sampling positions [S1, unofficial].
- **Alignment:** `align_corners=True` must be used. Using `False` (default in some versions) will shift coordinates by half a pixel, causing misalignment with the ray origins [S1, unofficial].
- **Shape Mismatch:** The `xy_grid` must be reshaped correctly to `(N, -1, 1, 2)`. If the batch dimension is not handled correctly, `grid_sample` will fail or produce incorrect results.

## Subtopics

### [SRNO3D Query RGB](22967-srno3d-query-rgb.md)
SRNO3D Query RGB is a feature aggregation method that samples 3D feature maps at query coordinates by weighting eight corner samples according to the relative volume of the sub-cells they represent, then adding a residual from bilinear upsampling of the input.

## Operating characteristics
- **Time Complexity:** $O(N \cdot H \cdot W \cdot C)$ where $N$ is batch size, $H, W$ are spatial dimensions of the grid (here 1xM), and $C$ is channels. Since the grid height is 1, it is effectively $O(N \cdot M \cdot C)$.
- **Memory Complexity:** $O(N \cdot H \cdot W \cdot C)$ for the input and output tensors.
- **Performance:** Highly optimized via PyTorch's C++ backend. Performance scales with batch size and number of samples.
- **Strengths:** Simple to implement, leverages existing optimized bilinear interpolation.
- **Limitations:** Specific to PyTorch. Requires careful handling of coordinate conventions (negation, alignment).

## Background
This method is part of the "l3d_assignment3" (Light Field 3D Assignment 3) by Shreyas120 [S1, unofficial]. It is used in the context of volume rendering assignments to sample image colors at ray-surface intersection points. The method relies on PyTorch's `grid_sample` function, which is a standard tool for spatial transformer networks and coordinate-based rendering.

## Sources
[S1] Shreyas120/l3d_assignment3, https://github.com/Shreyas120/l3d_assignment3, fetched 2026-10-26 (unofficial)
