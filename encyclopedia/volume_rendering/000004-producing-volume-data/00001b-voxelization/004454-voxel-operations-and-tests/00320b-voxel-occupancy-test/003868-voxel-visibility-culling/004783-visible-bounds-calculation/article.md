# Visible Bounds Calculation


## Overview
Visible Bounds Calculation determines the physical-space bounding box of voxels with composited opacity above a specific threshold, allowing the camera to frame only the visible content rather than the full volume extent [S1, unofficial]. It differs from sibling methods by using a downsampled, peak-based Gaussian approximation of opacity to achieve significant performance gains (from ~1-2s to ~10ms) while maintaining sufficient accuracy for camera framing [S1, unofficial].

## How it works
**Inputs:**
- Volume data (implicit or explicit, accessed via transfer function peaks).
- `N_PEAKS`: Number of transfer function peaks (one per class).
- `VISIBLE_BOUNDS_STRIDE`: Downsampling factor (default 4).
- `VISIBLE_BOUNDS_THRESHOLD`: Opacity threshold (default 0.05).
- `VISIBLE_BOUNDS_MARGIN`: Padding fraction (default 0.15).
- `spacing`: Physical voxel dimensions per axis.

**Outputs:**
- A bounding box defined by `[lo_x, lo_y, lo_z, hi_x, hi_y, hi_z]` in physical coordinates, or `None` if no voxels are visible.

**Data Structures:**
- `opacity_grid`: A 3D array of size `ceil(resolution / VISIBLE_BOUNDS_STRIDE)` initialized to zeros.

**Steps:**
1. **Downsample:** Create `opacity_grid` with dimensions `ceil(resolution / VISIBLE_BOUNDS_STRIDE)` on all axes.
2. **Initialize:** Set all values in `opacity_grid` to 0.0.
3. **Accumulate Opacity:** For each peak `i` in `N_PEAKS`:
   a. Retrieve peak parameters: `center` (HU value), `width` (standard deviation), and `height` (maximum opacity contribution).
   b. For each voxel index `sub` in the downsampled grid:
      i. Map `sub` to physical space or HU space as required by the transfer function. *Note: The source implies mapping to the domain of the Gaussian (likely HU space or normalized space).*
      ii. Calculate Gaussian contribution: $G = \exp(-0.5 \times ((sub - center) / width)^2)$.
      iii. Add to opacity: `opacity_grid[sub] += height * G`.
   c. *Clarification:* The source describes iterating peaks and adding to the grid. The exact mapping of `sub` to the Gaussian input is not fully detailed, but typically `sub` represents the position in the domain of the transfer function (e.g., HU values) or the voxel index mapped to physical space. Given the context of "peak parameters (center, width, height)", it is likely that the opacity is calculated per voxel based on its value (e.g., HU) against the peak. However, the pseudocode below assumes a simplified interpretation where the opacity is accumulated based on the peak's influence on the grid cells. *Correction:* The source says "For each peak i... Get peak parameters... Calculate Gaussian... Add height * Gaussian to opacity." This suggests the opacity grid is built by summing contributions from all peaks. The variable `sub` likely represents the voxel index or value. Given the performance note, it is likely a coarse grid. Let's assume `sub` iterates over the downsampled grid indices, and the Gaussian is evaluated based on the voxel's value (e.g., HU) at that location.
4. **Clip:** Clip all values in `opacity_grid` to the range `[0.0, 1.0]`.
5. **Mask:** Create a boolean mask where `opacity_grid > VISIBLE_BOUNDS_THRESHOLD`.
6. **Check Visibility:** If no voxels are `True` in the mask, return `None`.
7. **Find Extents:** For each axis (x, y, z):
   a. Find the minimum and maximum indices `min_idx` and `max_idx` where the mask is `True`.
8. **Convert to Physical:** Convert `min_idx` and `max_idx` to physical coordinates using `spacing`:
   - $lo_{phys} = min\_idx \times spacing$
   - $hi_{phys} = max\_idx \times spacing$
9. **Calculate Padding:** For each axis:
   - $extent = hi_{phys} - lo_{phys}$
   - $pad = \max(extent \times VISIBLE_BOUNDS_MARGIN, spacing \times VISIBLE_BOUNDS_STRIDE)$
10. **Extend Bounds:** Return `[lo_x - pad_x, lo_y - pad_y, lo_z - pad_z, hi_x + pad_x, hi_y + pad_y, hi_z + pad_z]`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_visible_bounds(resolution, spacing, peaks, stride=4, threshold=0.05, margin=0.15):
    """
    Calculate visible bounds of a volume based on opacity threshold.
    
    Args:
        resolution: Tuple (nx, ny, nz) of voxel counts.
        spacing: Tuple (sx, sy, sz) of physical voxel sizes.
        peaks: List of dicts with keys 'center', 'width', 'height'.
        stride: Downsampling factor.
        threshold: Opacity threshold.
        margin: Padding fraction.
        
    Returns:
        Tuple (lo_x, lo_y, lo_z, hi_x, hi_y, hi_z) or None.
    """
    # 1. Downsample
    new_res = tuple((r + stride - 1) // stride for r in resolution)
    opacity_grid = [[[[0.0 for _ in range(new_res[2])] for _ in range(new_res[1])] for _ in range(new_res[0])]
    
    # 2. Initialize (already done above)
    
    # 3. Accumulate Opacity
    # Note: The source implies iterating peaks and adding to the grid.
    # The exact mapping of 'sub' to the Gaussian input is ambiguous.
    # Assuming 'sub' refers to the voxel index in the downsampled grid,
    # and we need to map it to the domain of the transfer function (e.g., HU).
    # However, without explicit mapping, we assume the Gaussian is evaluated
    # based on the voxel's value. For simplicity, let's assume the volume
    # data is accessible or the peaks are evaluated per voxel.
    # Since the source mentions "For each peak i... Get peak parameters...
    # Calculate Gaussian... Add height * Gaussian to opacity", it suggests
    # a direct accumulation. Let's assume we have access to voxel values
    # or the Gaussian is precomputed.
    # Given the performance constraint, it's likely a coarse approximation.
    # We will assume a simplified model where the opacity is calculated
    # based on the peak's influence on the grid cells.
    
    # Re-reading source: "For each peak i in N_PEAKS: a. Get peak parameters...
    # b. Calculate Gaussian: exp(-0.5 * ((sub - center) / width)^2).
    # c. Add height * Gaussian to opacity."
    # This implies 'sub' is the variable being iterated. It is likely the
    # voxel index or value. Let's assume 'sub' is the voxel index in the
    # downsampled grid, and we map it to a physical or HU value.
    # For the sake of implementation, we assume we can get the voxel value
    # at each downsampled position.
    
    for peak in peaks:
        center = peak['center']
        width = peak['width']
        height = peak['height']
        
        for x in range(new_res[0]):
            for y in range(new_res[1]):
                for z in range(new_res[2]):
                    # Map downsampled index to physical/HU value
                    # This step is not fully detailed in the source.
                    # Assuming a linear mapping from index to HU value for simplicity.
                    # In practice, this would involve looking up the voxel value.
                    voxel_value = get_voxel_value(x * stride, y * stride, z * stride)
                    
                    # Calculate Gaussian
                    gaussian = math.exp(-0.5 * ((voxel_value - center) / width) ** 2)
                    
                    # Add to opacity
                    opacity_grid[x][y][z] += height * gaussian
    
    # 4. Clip
    for x in range(new_res[0]):
        for y in range(new_res[1]):
            for z in range(new_res[2]):
                opacity_grid[x][y][z] = max(0.0, min(1.0, opacity_grid[x][y][z]))
    
    # 5. Mask
    mask = [[[[opacity_grid[x][y][z] > threshold for z in range(new_res[2])] for y in range(new_res[1])] for x in range(new_res[0])]
    
    # 6. Check Visibility
    if not any(any(any(mask[x][y][z] for z in range(new_res[2])) for y in range(new_res[1])) for x in range(new_res[0])):
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

def get_voxel_value(x, y, z):
    # Placeholder for actual voxel value retrieval
    # In a real implementation, this would access the volume data
    return 0.0 # Not given in the sources.
```

**Parameters:**
- `VISIBLE_BOUNDS_STRIDE`: 4. Reason: Full resolution takes ~1-2s, strided takes ~10ms [S1, unofficial].
- `VISIBLE_BOUNDS_THRESHOLD`: 0.05 [S1, unofficial].
- `VISIBLE_BOUNDS_MARGIN`: 0.15 (15% of extent). Minimum padding is one strided voxel spacing [S1, unofficial].

**Edge Cases and Pitfalls:**
- If no voxels exceed the opacity threshold, the function returns `None`, causing the caller to fall back to full-volume framing [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N / S^3)$ where $N$ is the total number of voxels and $S$ is the stride. The source reports ~10ms for a 523-slice body scan with striding, compared to ~1-2s for full resolution [S1, unofficial].
- **Memory Complexity:** $O(N / S^3)$ for the downsampled opacity grid.
- **Performance:** 57-58 fps head-on and 84-88 fps while rotating on a 523-slice body scan at the same resolution [S1, unofficial].
- **Strengths:** Significant performance improvement over full-resolution opacity calculation.
- **Limitations:** Approximation may miss small visible structures if they fall below the threshold or are obscured by the downsampling.
- **When to use:** When camera framing needs to be dynamic and responsive, and full-volume framing is too coarse or performance is constrained.
- **When not to use:** When precise visibility is required for small structures or when the transfer function is highly sparse and striding might miss peaks.

## Background
This method is part of a reinforcement learning-based volume rendering transfer function optimization system, specifically designed for VR applications [S1, unofficial]. It was developed to address the need for efficient camera framing in interactive volume rendering, where full-volume framing can be inefficient or visually misleading [S1, unofficial].

## Sources
[S1] kevinkunkel98/reinforcement-learning-volume-rendering-transfer-function, https://github.com/kevinkunkel98/reinforcement-learning-volume-rendering-transfer-function, fetched 2026-10-27 (unofficial)
