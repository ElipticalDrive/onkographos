# Downscaled Grid Map Generation

## Overview
Generates a grid map for a downscaled resolution and an index map to map high-resolution coordinates to low-resolution coordinates [S1, unofficial].
## How it works
**Inputs:**
- Original grid map dimensions: $H_{orig}, W_{orig}$ (height and width) [S1, unofficial].
- Scale factor: $s$ (downsampling ratio) [S1, unofficial].
- Stride: $stride$ (spatial step size of the original grid) [S1, unofficial].

**Outputs:**
- `grid_map`: A tensor of shape $(H_{new}, W_{new}, 2)$ containing scaled coordinates [S1, unofficial].
- `index_map`: A tensor of shape $(H_{orig}, W_{orig})$ containing integer indices mapping each high-resolution coordinate to a low-resolution position [S1, unofficial].

**Data Structures:**
- `grid_map`: Float tensor. Each element $(i, j)$ holds the scaled 2D coordinate for the downscaled grid [S1, unofficial].
- `index_map`: Integer tensor. Each element $(h, w)$ holds the flattened index into the new grid map corresponding to the high-resolution coordinate $(h, w)$ [S1, unofficial].

**Steps:**
1. Calculate the stride of the original grid map [S1, unofficial].
2. Compute new height and width by dividing original dimensions by scale and taking the ceiling:
   $$ H_{new} = \lceil H_{orig} / s \rceil $$
   $$ W_{new} = \lceil W_{orig} / s \rceil $$ [S1, unofficial].
3. Generate a new meshgrid for the downscaled resolution [S1, unofficial].
4. Scale the new grid map by `scale * stride` and shift it by `((scale * stride) - 1) / 2` [S1, unofficial]:
   $$ grid\_map[i, j, 0] = i \cdot s \cdot stride + \frac{s \cdot stride - 1}{2} $$
   $$ grid\_map[i, j, 1] = j \cdot s \cdot stride + \frac{s \cdot stride - 1}{2} $$ [S1, unofficial].
5. Compute the index map by dividing the original grid mesh coordinates by scale, adding 1, taking the ceiling, and subtracting 1 [S1, unofficial]:
   $$ row\_idx[h, w] = \lceil h / s + 1 \rceil - 1 $$
   $$ col\_idx[h, w] = \lceil w / s + 1 \rceil - 1 $$
   $$ index\_map[h, w] = row\_idx[h, w] \times W_{new} + col\_idx[h, w] $$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def generate_downscaled_grid_map(H_orig, W_orig, scale, stride=1.0):
    # Step 1: Calculate stride (assumed constant for the grid)
    # Step 2: Compute new dimensions
    H_new = math.ceil(H_orig / scale)
    W_new = math.ceil(W_orig / scale)

    # Step 3: Generate a new meshgrid for the downscaled resolution
    i_coords = np.arange(H_new)
    j_coords = np.arange(W_new)
    I, J = np.meshgrid(i_coords, j_coords, indexing='ij')

    # Step 4: Scale the new grid map and shift
    scale_factor = scale * stride
    shift = (scale_factor - 1.0) / 2.0
    grid_map = np.zeros((H_new, W_new, 2))
    grid_map[:, :, 0] = I * scale_factor + shift
    grid_map[:, :, 1] = J * scale_factor + shift

    # Step 5: Compute the index map
    # Generate meshgrid for the original high-resolution coordinates
    h_coords = np.arange(H_orig)
    w_coords = np.arange(W_orig)
    H_mesh, W_mesh = np.meshgrid(h_coords, w_coords, indexing='ij')

    # Divide by scale, add 1, take ceiling, subtract 1
    row_idx = np.ceil(H_mesh / scale + 1).astype(int) - 1
    col_idx = np.ceil(W_mesh / scale + 1).astype(int) - 1

    # Flatten to 1D indices into the new grid map
    index_map = row_idx * W_new + col_idx

    # Pitfall check
    if np.max(index_map) >= H_new * W_new:
        raise ValueError("Maximum index in the flattened index map exceeds the number of elements in the new grid map.")

    return grid_map, index_map
```

**Parameters:**
- `scale`: Typical value depends on the desired downsampling factor (e.g., 2 for 2x downsampling) [S1, unofficial].
- `stride`: Typically 1.0 for normalized coordinates [S1, unofficial].

**Edge Cases and Pitfalls:**
- If the maximum index in the flattened index map exceeds the number of elements in the new grid map, a `ValueError` is raised [S1, unofficial]. This occurs when the ceiling operation pushes indices out of bounds due to rounding or scale factors [S1, unofficial].

## Subtopics

### [Index-Based Upsampling with Recalibration](2352-index-based-upsampling-with-recalibration.md)
Upsamples low-resolution tensors to high resolution by gathering values via an index map and scaling them with a recalibration map, specifically designed for depth values. It differs from sibling methods like Depth-Aware Upsampling by relying on explicit index and scaling tensors rather than depth-weighted interpolation.

## Operating characteristics
- **Time Complexity:** $O(H_{new} \cdot W_{new})$ for generating the grid map and index map [G].
- **Memory Complexity:** $O(H_{new} \cdot W_{new})$ for storing the grid map and index map [G].
- **Strengths:** Provides explicit coordinate mappings which can be useful for differentiable rendering or precise upsampling [S1, unofficial].
- **Limitations:** The method is specific to the coordinate generation and does not handle value aggregation [S1, unofficial]. The pitfall regarding index bounds requires careful handling of scale factors and grid dimensions [S1, unofficial].

## Background
This method is described in the context of a differentiable renderer for optimizing shapes from 2D observations [S1, unofficial]. It is part of the DIST-Renderer project [S1, unofficial].

## Sources
[S1] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
