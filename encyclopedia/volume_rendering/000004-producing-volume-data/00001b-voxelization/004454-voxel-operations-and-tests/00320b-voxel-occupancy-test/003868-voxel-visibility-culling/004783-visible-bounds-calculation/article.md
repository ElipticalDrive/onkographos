# Visible Bounds Calculation


## Overview
Visible Bounds Calculation determines the physical-space bounding box of voxels with composited opacity above a threshold, framing the camera on visible content rather than the full volume extent [S1, unofficial]. It differs from sibling methods by using a downsampled, peak-based Gaussian approximation of opacity to achieve significant performance gains while maintaining sufficient accuracy for camera framing [S1, unofficial].

## How it works
**Inputs:**
- `resolution`: Tuple `(nx, ny, nz)` of voxel counts per axis.
- `spacing`: Tuple `(sx, sy, sz)` of physical voxel dimensions per axis.
- `peaks`: List of peak parameters, each containing `center` (HU value), `width` (standard deviation), and `height` (maximum opacity contribution).
- `VISIBLE_BOUNDS_STRIDE`: Downsampling factor per axis (default 4) [S1, unofficial].
- `VISIBLE_BOUNDS_THRESHOLD`: Opacity threshold to consider a voxel visible (default 0.05) [S1, unofficial].
- `VISIBLE_BOUNDS_MARGIN`: Padding fraction added to bounds (default 0.15) [S1, unofficial].
- `volume_data`: 3D array of voxel values (e.g., HU) accessed at strided intervals.

**Outputs:**
- A bounding box defined by `[lo_x, lo_y, lo_z, hi_x, hi_y, hi_z]` in physical coordinates, or `None` if no voxels are visible [S1, unofficial].

**Data Structures:**
- `opacity_grid`: 3D array of size `ceil(resolution / VISIBLE_BOUNDS_STRIDE)` initialized to zeros.

**Steps:**
1. **Downsample:** Compute `new_res = ceil(resolution / VISIBLE_BOUNDS_STRIDE)` on all axes [S1, unofficial].
2. **Initialize:** Set all values in `opacity_grid` to 0.0 [S1, unofficial].
3. **Accumulate Opacity:** For each peak in `peaks`:
   a. Retrieve peak parameters: `center`, `width`, `height` [S1, unofficial].
   b. For each voxel index `(x, y, z)` in `opacity_grid`:
      i. Sample `volume_data` at `(x * stride, y * stride, z * stride)` to get the voxel value `sub` [G].
      ii. Calculate Gaussian contribution: $G = \exp(-0.5 \times ((sub - center) / width)^2)$ [S1, unofficial].
      iii. Add to opacity: `opacity_grid[x, y, z] += height * G` [S1, unofficial].
4. **Clip:** Clip all values in `opacity_grid` to the range `[0.0, 1.0]` [S1, unofficial].
5. **Mask:** Create a boolean mask where `opacity_grid > VISIBLE_BOUNDS_THRESHOLD` [S1, unofficial].
6. **Check Visibility:** If no voxels are `True` in the mask, return `None` [S1, unofficial].
7. **Find Extents:** For each axis `(x, y, z)`: find the minimum and maximum indices `min_idx` and `max_idx` where the mask is `True` [S1, unofficial].
8. **Convert to Physical:** Convert `min_idx` and `max_idx` to physical coordinates using `spacing`:
   - $lo_{phys} = min\_idx \times spacing$
   - $hi_{phys} = max\_idx \times spacing$ [S1, unofficial].
9. **Calculate Padding:** For each axis:
   - $extent = hi_{phys} - lo_{phys}$
   - $pad = \max(extent \times VISIBLE_BOUNDS_MARGIN, spacing \times VISIBLE_BOUNDS_STRIDE)$ [S1, unofficial].
10. **Extend Bounds:** Return `[lo_x - pad_x, lo_y - pad_y, lo_z - pad_z, hi_x + pad_x, hi_y + pad_y, hi_z + pad_z]` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def calculate_visible_bounds(resolution, spacing, peaks, volume_data,
                             stride=4, threshold=0.05, margin=0.15):
    """Calculate visible bounds of a volume based on opacity threshold."""
    # 1. Downsample
    new_res = tuple((r + stride - 1) // stride for r in resolution)
    
    # 2. Initialize
    opacity_grid = [[[0.0 for _ in range(new_res[2])] 
                     for _ in range(new_res[1])] 
                    for _ in range(new_res[0])]
    
    # 3. Accumulate Opacity
    for peak in peaks:
        center = peak['center']
        width = peak['width']
        height = peak['height']
        for x in range(new_res[0]):
            for y in range(new_res[1]):
                for z in range(new_res[2]):
                    # Sample voxel value at strided position
                    sub = volume_data[x * stride][y * stride][z * stride]
                    # Calculate Gaussian contribution
                    gaussian = math.exp(-0.5 * ((sub - center) / width) ** 2)
                    # Add to opacity
                    opacity_grid[x][y][z] += height * gaussian
    
    # 4. Clip
    for x in range(new_res[0]):
        for y in range(new_res[1]):
            for z in range(new_res[2]):
                opacity_grid[x][y][z] = max(0.0, min(1.0, opacity_grid[x][y][z]))
    
    # 5. Mask
    mask = [[[opacity_grid[x][y][z] > threshold for z in range(new_res[2])] 
             for y in range(new_res[1])] 
            for x in range(new_res[0])]
    
    # 6. Check Visibility
    if not any(any(any(mask[x][y][z] for z in range(new_res[2])) 
                   for y in range(new_res[1])) 
               for x in range(new_res[0])):
        return None
    
    # 7. Find Extents
    bounds = []
    for axis in range(3):
        min_idx = None
        max_idx = None
        for i in range(new_res[axis]):
            # Check if any voxel in the other dimensions is True
            if axis == 0:
                if any(any(mask[i][y][z] for z in range(new_res[2])) for y in range(new_res[1])):
                    if min_idx is None: min_idx = i
                    max_idx = i
            elif axis == 1:
                if any(any(mask[x][i][z] for z in range(new_res[2])) for x in range(new_res[0])):
                    if min_idx is None: min_idx = i
                    max_idx = i
            elif axis == 2:
                if any(any(mask[x][y][i] for y in range(new_res[1])) for x in range(new_res[0])):
                    if min_idx is None: min_idx = i
                    max_idx = i
        bounds.append((min_idx, max_idx))
    
    # 8. Convert to Physical
    lo_phys = []
    hi_phys = []
    for axis in range(3):
        min_idx, max_idx = bounds[axis]
        lo_phys.append(min_idx * spacing[axis])
        hi_phys.append(max_idx * spacing[axis])
    
    # 9. Calculate Padding
    padded_bounds = []
    for axis in range(3):
        extent = hi_phys[axis] - lo_phys[axis]
        pad = max(extent * margin, spacing[axis] * stride)
        padded_bounds.append(lo_phys[axis] - pad)
        padded_bounds.append(hi_phys[axis] + pad)
    
    return tuple(padded_bounds)
```

**Parameters:**
- `VISIBLE_BOUNDS_STRIDE`: 4. Reason: Full resolution takes ~1-2s, strided takes ~10ms [S1, unofficial].
- `VISIBLE_BOUNDS_THRESHOLD`: 0.05 [S1, unofficial].
- `VISIBLE_BOUNDS_MARGIN`: 0.15 (15% of extent). Minimum padding is one strided voxel spacing [S1, unofficial].

**Edge Cases and Pitfalls:**
- If no voxels exceed the opacity threshold, the function returns `None`, causing the caller to fall back to full-volume framing [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N / S^3)$ where $N$ is the total number of voxels and $S$ is the stride [G].
- **Memory Complexity:** $O(N / S^3)$ for the downsampled opacity grid [G].
- **Performance:** 57-58 fps head-on and 84-88 fps while rotating on a 523-slice body scan at the same resolution [S1, unofficial].
- **Strengths:** Significant performance improvement over full-resolution opacity calculation [S1, unofficial].
- **Limitations:** Not given in the sources.
- **When to use:** When camera framing needs to be dynamic and responsive, and full-volume framing is too coarse or performance is constrained [G].
- **When not to use:** When precise visibility is required for small structures or when the transfer function is highly sparse and striding might miss peaks [G].

## Background
This method is part of a volume rendering transfer function optimization system designed for VR applications [S1, unofficial]. It was developed to address the need for efficient camera framing in interactive volume rendering, where full-volume framing can be inefficient or visually misleading [S1, unofficial].

## Sources
[S1] kevinkunkel98/reinforcement-learning-volume-rendering-transfer-function, https://github.com/kevinkunkel98/reinforcement-learning-volume-rendering-transfer-function, fetched 2026-10-27 (unofficial)
