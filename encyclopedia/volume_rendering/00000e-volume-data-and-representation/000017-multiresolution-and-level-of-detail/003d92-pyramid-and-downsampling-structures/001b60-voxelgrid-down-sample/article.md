# VoxelGrid Down Sample
## Overview
VoxelGrid Down Sample reduces a volumetric voxel grid's resolution by an integer factor using max pooling [S1, unofficial]. It differs from [Pre-processing for Half Resolution](6436-pre-processing-for-half-resolution.md), which fills transparent holes with median values, by strictly applying block-wise maximums without hole-filling [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A `VoxelGrid` object containing a 3D data array `data` of shape $(N_x, N_y, N_z)$ and spatial metadata (`loc`, `scale`) [S1, unofficial].
- Parameter: An integer downsampling `factor` $f$.
- Output: A new `VoxelGrid` object with resolution $(N_x/f, N_y/f, N_z/f)$ and the same `loc` and `scale` [S1, unofficial].

**Data Structures:**
- `VoxelGrid`: Contains `data` (numeric array) and spatial attributes `loc`, `scale` [S1, unofficial].

**Steps:**
1. **Validate Divisibility:** Check if the current resolution dimensions are divisible by the `factor`. If $N_x \% f \neq 0$ or $N_y \% f \neq 0$ or $N_z \% f \neq 0$, raise a `ValueError` [S1, unofficial].
2. **Downsample Data:** Apply max pooling over non-overlapping $f \times f \times f$ blocks. The value of each new voxel is the maximum value found in the corresponding block of the input grid [S1, unofficial].
   $$ V_{\text{new}}(i, j, k) = \max_{\substack{0 \le dx < f \\ 0 \le dy < f \\ 0 \le dz < f}} V_{\text{old}}(i \cdot f + dx, j \cdot f + dy, k \cdot f + dz) $$
   for $0 \le i < N_x/f$, $0 \le j < N_y/f$, $0 \le k < N_z/f$.
3. **Construct Output:** Create a new `VoxelGrid` instance using the downsampled data array and copy the original `loc` and `scale` attributes [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class VoxelGrid:
    def __init__(self, data, loc=None, scale=None):
        self.data = data
        self.loc = loc
        self.scale = scale

def voxelgrid_down_sample(voxel_grid, factor):
    data = voxel_grid.data
    shape = data.shape
    if not (shape[0] % factor == 0 and shape[1] % factor == 0 and shape[2] % factor == 0):
        raise ValueError(f"Resolution {shape} is not divisible by factor {factor}")
    new_shape = (shape[0] // factor, factor, shape[1] // factor, factor, shape[2] // factor, factor)
    reshaped = data.reshape(new_shape)
    downsampled_data = reshaped.max(axis=(1, 3, 5))
    return VoxelGrid(data=downsampled_data, loc=voxel_grid.loc, scale=voxel_grid.scale)
```

**Parameters:**
- `factor`: Integer downsampling factor. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Non-divisible Resolution:** If the grid dimensions are not multiples of the factor, the method raises a `ValueError` [S1, unofficial].
- **Max Pooling Artifacts:** Max pooling preserves high-intensity features but discards low-intensity information [S1, unofficial].

## Subtopics

### Max-Pooling Layer
A layer that applies a max filter to non-overlapping sub-regions of the input to downsample the input and reduce parameters in subsequent encoder layers [S2, unofficial]. It contains no trainable parameters [S2, unofficial]. Typical parameters include a filter size of $2 \times 2$ and a stride of 2, applied in layers with input channels 64, 32, and 8 [S2, unofficial].

## Operating characteristics
- Time complexity: $O(N)$, where $N$ is the number of input voxels, as each voxel is processed once [G].
- Memory complexity: $O(N/f^3)$ for the output grid, plus $O(N)$ temporary memory for reshaping [G].
- Limitations: Requires grid dimensions to be exact multiples of the factor [S1, unofficial]. Discards non-maximum values within each block [S1, unofficial].

## Background
Not given in the sources.

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S2] Compressed volume rendering using deep learning, https://www.cs.umd.edu/sites/default/files/scholarly_papers/Jain.pdf, fetched 2026-10-26 (unofficial)
