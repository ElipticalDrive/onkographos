# Vectorized Voxelization

Also called: Vectorized Point Cloud Voxelization

## Overview
Vectorized Voxelization is a vectorized algorithm that bins 3D points into a voxel grid, averaging colors per cell, and returns voxel centers, averaged colors, and the grid origin [S1, unofficial].

## How it works
**Inputs:**
- `positions`: Array of shape $(N, 3)$, dtype `float32`, representing 3D coordinates of $N$ points [S1, unofficial].
- `colors`: Optional array of shape $(N, 3)$, dtype `uint8` (0–255) or `float32` (0–1) [S1, unofficial]. If `None`, a grayscale height map is generated [S1, unofficial].
- `opacities`: Optional array of shape $(N,)$, dtype `float32` [S1, unofficial].
- `resolution`: Float, the side length of each voxel cube [S1, unofficial].
- `origin`: Optional array of shape $(3,)$, dtype `float32` [S1, unofficial]. If `None`, computed as `positions.min(axis=0)` [S1, unofficial].
- `min_opacity`: Float, default `0.0` [S1, unofficial]. Points with opacity below this are discarded [S1, unofficial].
- `density_threshold`: Int, default `1` [S1, unofficial]. Voxels with fewer points than this are discarded [S1, unofficial].

**Outputs:**
- `centers`: Array of shape $(M, 3)$, dtype `float32`, representing the 3D coordinates of the $M$ active voxel centers [S1, unofficial].
- `colors`: Array of shape $(M, 3)$, dtype `float32`, representing the averaged colors in the range $[0, 1]$ [S1, unofficial].
- `origin`: Array of shape $(3,)$, dtype `float32`, the grid origin used for quantization [S1, unofficial].

**Data Structures:**
- `grid`: Array of shape $(N, 3)$, dtype `int64`, storing the integer voxel indices for each point [S1, unofficial].
- `keys`: Array of shape $(N,)$, dtype `int64`, storing the flattened 1D index for each voxel [S1, unofficial].
- `unique_keys`: Array of shape $(K,)$, dtype `int64`, storing the sorted, unique voxel keys [S1, unofficial].
- `counts`: Array of shape $(K,)$, dtype `int64`, storing the number of points per unique voxel [S1, unofficial].

**Steps:**

1.  **Opacity Filtering:** If `opacities` is provided and `min_opacity > 0.0`, filter `positions`, `colors`, and `opacities` to keep only indices where `opacities >= min_opacity` [S1, unofficial]. If all points are removed, raise `ValueError: 'Every point was removed by the opacity filter - lower Min opacity'` [S1, unofficial].

2.  **Fallback Color Generation:** If `colors` is `None`, compute a grayscale color based on the height ($z$-coordinate) of the points:
    $$ z = \text{positions}[:, 2] $$
    $$ t = \frac{z - \min(z)}{\max(\text{ptp}(z), 10^{-8})} $$
    $$ \text{colors} = \text{repeat}\left( \left( t \times 255 \right).astype(\text{uint8}), 3, \text{axis}=1 \right) $$
    where $\text{ptp}(z) = \max(z) - \min(z)$ [S1, unofficial].

3.  **Origin Initialization:** If `origin` is `None`, set `origin = positions.min(axis=0)` and cast `origin` to `np.float32` [S1, unofficial].

4.  **Quantization:** Compute the integer grid coordinates for each point:
    $$ \text{grid} = \left\lfloor \frac{\text{positions} - \text{origin}}{\text{resolution}} \right\rfloor.astype(\text{int64}) $$
    [S1, unofficial].

5.  **Non-Negative Shift:** Ensure all grid indices are non-negative for hashing:
    $$ \text{grid\_offset} = \text{grid.min(axis=0)} $$
    $$ \text{grid} \leftarrow \text{grid} - \text{grid\_offset} $$
    [S1, unofficial].

6.  **Grid Dimensions:** Compute the dimensions of the bounding box in voxels:
    $$ \text{dims} = \text{grid.max(axis=0)} + 1 $$
    [S1, unofficial].

7.  **Flattening:** Collapse 3D indices $(gx, gy, gz)$ into a single 1D key using row-major ordering:
    $$ \text{keys} = (\text{grid}[:, 0] \times \text{dims}[1] + \text{grid}[:, 1]) \times \text{dims}[2] + \text{grid}[:, 2] $$
    [S1, unofficial].

8.  **Sorting:** Sort the keys to group identical voxels together:
    $$ \text{order} = \text{argsort}(\text{keys}) $$
    $$ \text{keys\_sorted} = \text{keys}[\text{order}] $$
    [S1, unofficial].

9.  **Unique Identification:** Find unique voxel keys and their first occurrence indices and counts:
    $$ \text{unique\_keys}, \text{start}, \text{counts} = \text{unique}(\text{keys\_sorted}, \text{return\_index}=\text{True}, \text{return\_counts}=\text{True}) $$
    [S1, unofficial].

10. **Color Aggregation:** Compute the sum of colors for each unique voxel using cumulative sums on the sorted colors:
    $$ \text{colors\_sorted} = \text{colors}[\text{order}] $$
    $$ \text{csum} = \text{cumsum}(\text{colors\_sorted.astype}(\text{float64}), \text{axis}=0) $$
    Prepend a zero row to `csum`:
    $$ \text{csum} = \text{vstack}([\text{zeros}(3), \text{csum}]) $$
    Compute sums at the boundaries of each unique group:
    $$ \text{color\_sums} = \text{csum}[\text{start} + \text{counts}] - \text{csum}[\text{start}] $$
    [S1, unofficial].

11. **Averaging:** Compute the average color per voxel and normalize to $[0, 1]$:
    $$ \text{avg\_colors} = \left( \frac{\text{color\_sums}}{\text{counts}[:, \text{None}]} \right).astype(\text{float32}) / 255.0 $$
    [S1, unofficial].

12. **Density Filtering:** If `density_threshold > 1`, filter out voxels with insufficient points:
    $$ \text{keep} = \text{counts} \ge \text{density\_threshold} $$
    $$ \text{unique\_keys} \leftarrow \text{unique\_keys}[\text{keep}] $$
    $$ \text{avg\_colors} \leftarrow \text{avg\_colors}[\text{keep}] $$
    [S1, unofficial].

13. **Center Reconstruction:** Reconstruct the 3D coordinates of the voxel centers from the flattened keys:
    $$ gz = \text{unique\_keys} \% \text{dims}[2] $$
    $$ gy = (\text{unique\_keys} // \text{dims}[2]) \% \text{dims}[1] $$
    $$ gx = \text{unique\_keys} // (\text{dims}[1] \times \text{dims}[2]) $$
    $$ \text{centers} = \text{origin} + (\text{stack}([gx, gy, gz]).T + 0.5) \times \text{resolution} $$
    The standard interpretation for voxel centers is the grid index plus 0.5, scaled by resolution, added to the origin [G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def vectorized_voxelization(positions, colors=None, opacities=None, 
                            resolution=0.05, origin=None, 
                            min_opacity=0.0, density_threshold=1):
    """
    Vectorized voxelization of a point cloud.
    
    Args:
        positions: (N, 3) float32 array of point coordinates.
        colors: (N, 3) uint8 array (0-255) or None.
        opacities: (N,) float32 array or None.
        resolution: float, voxel side length.
        origin: (3,) float32 array or None.
        min_opacity: float, minimum opacity threshold.
        density_threshold: int, minimum points per voxel.
        
    Returns:
        centers: (M, 3) float32 array of voxel centers.
        colors: (M, 3) float32 array of averaged colors (0-1).
        origin: (3,) float32 array of grid origin.
    """
    # 1. Opacity Filtering
    if opacities is not None and min_opacity > 0.0:
        mask = opacities >= min_opacity
        if not np.any(mask):
            raise ValueError('Every point was removed by the opacity filter - lower Min opacity')
        positions = positions[mask]
        if colors is not None:
            colors = colors[mask]
        opacities = opacities[mask]
        
    # 2. Fallback Color Generation
    if colors is None:
        z = positions[:, 2]
        z_min = np.min(z)
        z_range = np.ptp(z)
        if z_range < 1e-8:
            z_range = 1e-8
        t = (z - z_min) / z_range
        colors = np.repeat((t[:, None] * 255).astype(np.uint8), 3, axis=1)
        
    # 3. Origin Initialization
    if origin is None:
        origin = np.min(positions, axis=0).astype(np.float32)
    else:
        origin = origin.astype(np.float32)
        
    # 4. Quantization
    grid = np.floor((positions - origin) / resolution).astype(np.int64)
    
    # 5. Non-Negative Shift
    grid_offset = np.min(grid, axis=0)
    grid -= grid_offset
    
    # 6. Grid Dimensions
    dims = np.max(grid, axis=0) + 1
    
    # 7. Flattening
    keys = (grid[:, 0] * dims[1] + grid[:, 1]) * dims[2] + grid[:, 2]
    
    # 8. Sorting
    order = np.argsort(keys)
    keys_sorted = keys[order]
    
    # 9. Unique Identification
    unique_keys, start, counts = np.unique(keys_sorted, return_index=True, return_counts=True)
    
    # 10. Color Aggregation
    colors_sorted = colors[order].astype(np.float64)
    csum = np.cumsum(colors_sorted, axis=0)
    csum = np.vstack([np.zeros(3), csum])
    color_sums = csum[start + counts] - csum[start]
    
    # 11. Averaging
    avg_colors = (color_sums / counts[:, None]).astype(np.float32) / 255.0
    
    # 12. Density Filtering
    if density_threshold > 1:
        keep = counts >= density_threshold
        unique_keys = unique_keys[keep]
        avg_colors = avg_colors[keep]
        
    # 13. Center Reconstruction
    gz = unique_keys % dims[2]
    gy = (unique_keys // dims[2]) % dims[1]
    gx = unique_keys // (dims[1] * dims[2])
    
    centers = origin + (np.stack([gx, gy, gz], axis=-1).astype(np.float32) + 0.5) * resolution
    
    return centers, avg_colors, origin
```

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Not given in the sources.
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.

## Background
- **Origin:** The method originates from the `3dgs-voxelizer-Omni` repository [S1, unofficial].
- **History and development:** Not given in the sources.
- **Usage:** Not given in the sources.

## Sources
[S1] omolism/3dgs-voxelizer-Omni, https://github.com/omolism/3dgs-voxelizer-Omni, fetched 2026-10-27 (unofficial)
