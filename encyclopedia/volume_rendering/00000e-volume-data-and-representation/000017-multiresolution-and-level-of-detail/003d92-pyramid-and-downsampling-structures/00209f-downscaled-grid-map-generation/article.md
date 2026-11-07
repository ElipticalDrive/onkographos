# Downscaled Grid Map Generation
## Overview
Downscaled Grid Map Generation creates a low-resolution coordinate grid and an index map to map high-resolution coordinates to their corresponding low-resolution locations [S1, unofficial].
## How it works
**Inputs:**
- Original grid map dimensions (height $H$, width $W$) [S1, unofficial].
- Scale factor $s$ [S1, unofficial].
- Stride $k$ of the original grid map [S1, unofficial].

**Outputs:**
- `new_grid_map`: A meshgrid for the downscaled resolution [S1, unofficial].
- `index_map`: A map where each element contains the index in the original grid map corresponding to the downscaled coordinate [S1, unofficial].

**Data Structures:**
- `new_grid_map`: Tensor of shape $(H', W', 2)$, where $H' = \lceil H/s \rceil$ and $W' = \lceil W/s \rceil$ [S1, unofficial].
- `index_map`: Tensor of shape $(H', W')$, containing integer indices [S1, unofficial].

**Steps:**
1. Calculate the stride $k$ of the original grid map [S1, unofficial].
2. Compute the new dimensions:
   $$ H' = \lceil H / s \rceil $$
   $$ W' = \lceil W / s \rceil $$
   [S1, unofficial]
3. Generate a new meshgrid for the downscaled resolution $(H', W')$. Let this be $G_{new}$, where each element $G_{new}[y, x] = (x, y)$ for $y \in [0, H'-1], x \in [0, W'-1]$ [S1, unofficial].
4. Scale the new grid map by $s \cdot k$ and shift it:
   $$ G_{scaled}[y, x] = G_{new}[y, x] \cdot (s \cdot k) + \frac{(s \cdot k) - 1}{2} $$
   [S1, unofficial]
5. Compute the index map by dividing the original grid mesh coordinates by $s$, adding 1, taking the ceiling, and subtracting 1:
   $$ I[y, x] = \lceil \frac{G_{scaled}[y, x]}{s} + 1 \rceil - 1 $$
   [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import math

def generate_downscaled_grid_map(original_h, original_w, scale, stride):
    # Step 2: Compute new dimensions
    new_h = math.ceil(original_h / scale)
    new_w = math.ceil(original_w / scale)
    
    # Step 3: Generate new meshgrid for downscaled resolution
    x_coords = np.arange(new_w)
    y_coords = np.arange(new_h)
    xx, yy = np.meshgrid(x_coords, y_coords, indexing='xy')
    new_grid_map = np.stack([xx, yy], axis=-1).astype(np.float32)
    
    # Step 4: Scale and shift
    shift = ((scale * stride) - 1) / 2
    scaled_grid = new_grid_map * (scale * stride) + shift
    
    # Step 5: Compute index map
    index_map = np.ceil(scaled_grid / scale + 1) - 1
    index_map = index_map.astype(np.int64)
    
    # Pitfall check
    max_index = np.max(index_map)
    new_elements = new_h * new_w
    if max_index >= new_elements:
        raise ValueError("Maximum index in the flattened index map exceeds the number of elements in the new grid map.")
        
    return new_grid_map, index_map
```

**Parameters:**
- `scale`: Typical values depend on the desired downsampling factor (e.g., 2, 4) [S1, unofficial].
- `stride`: Depends on the original grid map's spacing [S1, unofficial].

**Edge Cases and Pitfalls:**
- If the maximum index in the flattened index map exceeds the number of elements in the new grid map, a `ValueError` is raised [S1, unofficial].
- Precision issues may arise if `scale` is not an integer or if `stride` is very small [G].
## Subtopics

### [Index-Based Upsampling with Recalibration](2352-index-based-upsampling-with-recalibration.md)
Upsamples low-resolution tensors to high resolution using an index map and a recalibration map, specifically for depth values. It gathers values from the low-resolution tensor using the index map and multiplies them by the recalibration map to adjust for resolution changes. Choose it when explicit coordinate-based upsampling with recalibration is required for depth data.

## Operating characteristics
**Time Complexity:** $O(H' \cdot W')$, where $H'$ and $W'$ are the new dimensions [G].
**Memory Complexity:** $O(H' \cdot W')$ for the index map and grid map [G].
**Measured performance:** Not given in the sources.
**Strengths:** Provides explicit coordinate mapping for resolution transformations [S1, unofficial].
**Limitations:** Requires careful handling of boundary conditions and index validation [G].
**When to use:** When optimizing shapes from 2D observations using differentiable renderers that require coordinate mapping between resolutions [S1, unofficial].
**When not to use:** For simple visualization or when standard interpolation is sufficient [G].
## Background
This method is used in the DIST-Renderer project for optimizing shapes from 2D observations [S1, unofficial]. It is part of a pipeline that uses differentiable rendering to refine 3D shapes [S1, unofficial].
## Sources
[S1] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
