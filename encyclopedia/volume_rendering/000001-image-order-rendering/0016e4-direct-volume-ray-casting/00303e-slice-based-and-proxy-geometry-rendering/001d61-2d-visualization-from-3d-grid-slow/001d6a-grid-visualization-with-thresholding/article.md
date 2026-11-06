# Grid Visualization with Thresholding

Also called: 3D Scatter Plot Visualization, Thresholded Grid Visualization

## Overview
This method visualizes a 3D scalar grid by filtering cells based on opacity and gradient magnitude thresholds, then rendering the surviving cells as a 3D scatter plot [S1, unofficial]. It explicitly applies thresholding and cleaning steps before plotting, differing from its parent [2D Visualization from 3D Grid (Slow)](1507-2d-visualization-from-3d-grid-slow.md), and operates on a CPU-side scatter plot representation rather than a GPU-based ray-casting table like its sibling [3D Table Computation on GPU](2083-3d-table-computation-on-gpu.md).

## How it works
**Inputs:**
- `grid_cells`: A tensor of shape $(N, 4)$ or similar, containing voxel data (typically RGB/RGBA or scalar + alpha) [S1, unofficial].
- `gradient_data`: (Optional) A tensor of gradient magnitudes for each cell [S1, unofficial].
- `threshold`: Float. Opacity threshold for visualization. Default: $0.0$ [S1, unofficial].
- `do_threshold`: Boolean. Whether to apply thresholding. Default: `True` [S1, unofficial].
- `clean_strays`: Boolean. Whether to remove stray voxels. Default: `False` [S1, unofficial].

**Data Structures:**
- The grid is represented as a tensor where each row corresponds to a voxel [S1, unofficial].
- Alpha values are used to determine visibility [S1, unofficial].

**Steps:**
1. Load the `grid_cells` tensor [S1, unofficial].
2. If `gradient_data` exists and thresholding is enabled, zero out cells where the gradient norm is below $3.0 \times 10^{-6}$ [S1, unofficial].
3. Clip all values to the range $[0, 1]$ [S1, unofficial].
4. If `do_threshold` is enabled, zero out alpha values below `threshold` and set non-zero alphas to $1.0$ [S1, unofficial].
5. If `clean_strays` is enabled, apply a noise removal convolution twice: first with kernel size $3$ and radius $1$, then with kernel size $7$ and radius $3$ [S1, unofficial].
6. Extract coordinates and RGB values for cells with alpha $> 0$ [S1, unofficial].
7. Plot the extracted points using a 3D plotting library such as Plotly or Matplotlib [S1, unofficial].

**Pitfalls:**
- Gradient-based thresholding uses a fixed threshold of $3.0 \times 10^{-6}$ for gradient norm, while visualization thresholding uses a configurable opacity threshold [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def grid_visualization_with_thresholding(grid_cells, gradient_data=None, threshold=0.0, do_threshold=True, clean_strays=False):
    """
    Visualizes grid cells by filtering based on opacity and gradient thresholds.
    
    Args:
        grid_cells: np.ndarray of shape (N, 4) with RGBA values.
        gradient_data: np.ndarray of shape (N,) with gradient magnitudes.
        threshold: float, opacity threshold.
        do_threshold: bool, whether to apply thresholding.
        clean_strays: bool, whether to remove stray voxels.
        
    Returns:
        points: np.ndarray of shape (M, 3) with coordinates of visible cells.
        colors: np.ndarray of shape (M, 3) with RGB values of visible cells.
    """
    # Step 2: Gradient filtering
    if gradient_data is not None and do_threshold:
        mask = gradient_data >= 3.0e-06
        grid_cells[~mask] = 0.0
        
    # Step 3: Clip values to [0, 1]
    grid_cells = np.clip(grid_cells, 0.0, 1.0)
    
    # Step 4: Opacity thresholding
    if do_threshold:
        alpha = grid_cells[:, 3]
        alpha[alpha < threshold] = 0.0
        alpha[alpha > 0] = 1.0
        grid_cells[:, 3] = alpha
        
    # Step 5: Noise cleaning
    if clean_strays:
        # [G] The sources do not specify the exact convolution algorithm.
        # A standard 3D box filter convolution would be applied here.
        pass
        
    # Step 6: Extract coordinates and RGB values for cells with alpha > 0
    visible_mask = grid_cells[:, 3] > 0
    colors = grid_cells[visible_mask, :3]
    
    # Coordinates are derived from voxel indices assuming a linear grid layout
    # [G] Mapping 1D indices to 3D coordinates requires grid dimensions.
    # Assuming a cubic grid of size D, coordinates are computed as:
    D = int(np.ceil(np.cbrt(grid_cells.shape[0])))
    indices = np.where(visible_mask)[0]
    x = indices % D
    y = (indices // D) % D
    z = indices // (D * D)
    points = np.stack([x, y, z], axis=1)
    
    return points, colors
```

## Subtopics

### Density Range Windowing
Restricts rendering to voxels whose density falls within a specified $[min, max]$ range, allowing selective visualization of structures [S2, unofficial]. Raising `min` drops semi-transparent structures (skin/foliage) and retains dense cores (bone); lowering `max` peels off densest structures [S2, unofficial]. Parameters are `m_densityMin` (default $0.0f$) and `m_densityMax` (default $1.0f$) which define the visible density interval [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Simple to implement; effective for sparse data; allows interactive exploration via threshold tuning [S1, unofficial].
- **Limitations:** Fixed gradient threshold may not generalize; cleaning step is heuristic; scatter plot rendering may be slow for very large datasets [S1, unofficial].
- **When to use:** When visualizing sparse volumetric data or when noise reduction is required before plotting [S1, unofficial].
- **When not to use:** When continuous volume rendering is needed; when performance is critical for large datasets [S1, unofficial].

## Background
This method is derived from implementations in the Plenoxels repository [S1, unofficial] and GPUGems Volume Rendering examples [S2, unofficial]. It represents a simple approach to volume visualization by converting voxel data into a point cloud [S1, unofficial].

## Sources
[S1] DanJbk/Plenoxels, https://github.com/DanJbk/Plenoxels, fetched 2026-10-27 (unofficial)
[S2] sho3la/GPUGemsVolumeRendering, https://github.com/sho3la/GPUGemsVolumeRendering, fetched 2026-10-26 (unofficial)
