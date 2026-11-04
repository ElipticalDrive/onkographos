# Grid Visualization with Thresholding

Also called: 3D Scatter Plot Visualization, Thresholded Grid Visualization

## Overview
This method visualizes a 3D scalar grid by filtering cells based on opacity and gradient magnitude thresholds, then rendering the surviving cells as a 3D scatter plot. It solves the problem of visualizing sparse or noisy volumetric data by discarding low-confidence or irrelevant voxels. It differs from its parent [2D Visualization from 3D Grid (Slow)](1507-2d-visualization-from-3d-grid-slow.md) by explicitly applying thresholding and cleaning steps before plotting, and from its sibling [3D Table Computation on GPU](2083-3d-table-computation-on-gpu.md) by operating on a CPU-side scatter plot representation rather than a GPU-based ray-casting table.

## How it works
**Inputs:**
- `grid_cells`: A tensor of shape $(N, 4)$ or similar, containing voxel data (typically RGB/RGBA or scalar + alpha).
- `gradient_data`: (Optional) A tensor of gradient magnitudes for each cell.
- `threshold`: Float. Opacity threshold for visualization. Default: $0.0$ [S1, unofficial].
- `do_threshold`: Boolean. Whether to apply thresholding. Default: `True` [S1, unofficial].
- `clean_strays`: Boolean. Whether to remove stray voxels. Default: `False` [S1, unofficial].

**Data Structures:**
- The grid is represented as a tensor where each row corresponds to a voxel.
- Alpha values are used to determine visibility.

**Steps:**
1. **Load Data:** Load the `grid_cells` tensor. [S1, unofficial]
2. **Gradient Filtering (if applicable):** If `gradient_data` exists and thresholding is enabled, zero out cells where the gradient norm is below $3.0 \times 10^{-6}$. [S1, unofficial]
3. **Clipping:** Clip all values in the grid to the range $[0, 1]$. [S1, unofficial]
4. **Opacity Thresholding:** If `do_threshold` is enabled:
   - Zero out alpha values below the `threshold`.
   - Set all non-zero alpha values to $1.0$. [S1, unofficial]
5. **Noise Cleaning (if applicable):** If `clean_strays` is enabled, apply `convolve_grid_to_remove_noise` twice:
   - First pass: kernel size $3$, radius $1$.
   - Second pass: kernel size $7$, radius $3$. [S1, unofficial]
6. **Extraction:** Extract coordinates and RGB values for all cells where the final alpha value is $> 0$. [S1, unofficial]
7. **Plotting:** Render the extracted points using a 3D plotting library such as Plotly or Matplotlib. [S1, unofficial]

**Pitfalls:**
- Gradient-based thresholding uses a fixed threshold of $3.0 \times 10^{-6}$ for gradient norm, which may not be suitable for all datasets [S1, unofficial].
- Visualization thresholding uses a configurable opacity threshold, which must be tuned per dataset [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def grid_visualization_with_thresholding(grid_cells, gradient_data=None, threshold=0.0, do_threshold=True, clean_strays=False):
    """
    Visualizes grid cells by filtering based on opacity and gradient thresholds.
    
    Args:
        grid_cells: np.ndarray of shape (N, 4) or (N, 3) with RGB/RGBA values.
        gradient_data: np.ndarray of shape (N,) with gradient magnitudes.
        threshold: float, opacity threshold.
        do_threshold: bool, whether to apply thresholding.
        clean_strays: bool, whether to remove stray voxels.
        
    Returns:
        points: np.ndarray of shape (M, 3) with coordinates of visible cells.
        colors: np.ndarray of shape (M, 3) with RGB values of visible cells.
    """
    # Step 1: Load grid cells tensor [S1, unofficial]
    # grid_cells is assumed to be loaded already
    
    # Step 2: Gradient filtering [S1, unofficial]
    if gradient_data is not None and do_threshold:
        # Zero out cells where gradient norm is below 3.0e-06
        mask = gradient_data >= 3.0e-06
        grid_cells[~mask] = 0
        
    # Step 3: Clip values to [0, 1] [S1, unofficial]
    grid_cells = np.clip(grid_cells, 0.0, 1.0)
    
    # Step 4: Opacity thresholding [S1, unofficial]
    if do_threshold:
        # Assuming alpha is the last channel or a separate channel
        # For simplicity, assume grid_cells is (N, 4) with RGBA
        if grid_cells.shape[1] == 4:
            alpha = grid_cells[:, 3]
            # Zero out alpha values below threshold
            alpha[alpha < threshold] = 0.0
            # Set non-zero alphas to 1
            alpha[alpha > 0] = 1.0
            grid_cells[:, 3] = alpha
        else:
            # If no alpha channel, assume all are visible or handle appropriately
            pass
            
    # Step 5: Noise cleaning [S1, unofficial]
    if clean_strays:
        # Apply convolve_grid_to_remove_noise twice
        # Note: This is a placeholder for the actual convolution logic
        # Pass 1: kernel 3, radius 1
        grid_cells = convolve_grid_to_remove_noise(grid_cells, kernel_size=3, radius=1)
        # Pass 2: kernel 7, radius 3
        grid_cells = convolve_grid_to_remove_noise(grid_cells, kernel_size=7, radius=3)
        
    # Step 6: Extract coordinates and RGB values for cells with alpha > 0 [S1, unofficial]
    if grid_cells.shape[1] == 4:
        visible_mask = grid_cells[:, 3] > 0
    else:
        visible_mask = np.ones(grid_cells.shape[0], dtype=bool)
        
    # Assuming coordinates are derived from indices or provided separately
    # For this example, we assume grid_cells contains RGB and we need to map indices to 3D coords
    # This part depends on the specific grid structure, which is not fully detailed in [S1, unofficial]
    # [G] Assuming a simple linear mapping for demonstration, or that coordinates are pre-computed
    # In a real implementation, coordinates would be derived from the voxel indices in the 3D grid
    points = np.indices(grid_cells.shape[0]).T # Placeholder for actual coordinate extraction
    colors = grid_cells[visible_mask, :3]
    points = points[visible_mask]
    
    # Step 7: Plot using Plotly or Matplotlib [S1, unofficial]
    # This step is external to the data processing
    return points, colors

def convolve_grid_to_remove_noise(grid_cells, kernel_size, radius):
    """
    Placeholder for noise removal convolution.
    [G] This function is not defined in the sources. A real implementation would
    involve a 3D convolution or morphological operation.
    """
    # For the purpose of this article, we return the grid unchanged
    # as the specific algorithm is not provided in the sources.
    return grid_cells
```

## Subtopics

### Density Range Windowing
Restricts rendering to voxels whose density falls within a specified $[min, max]$ range, allowing selective visualization of structures [S2, unofficial]. To raise `min` drops semi-transparent structures (skin/foliage) and retains dense cores (bone); lowering `max` peels off densest structures [S2, unofficial]. Parameters are `m_densityMin` (default $0.0f$) and `m_densityMax` (default $1.0f$) which define the visible density interval [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Not given in the sources.
- **Strengths:** Simple to implement; effective for sparse data; allows interactive exploration via threshold tuning.
- **Limitations:** Fixed gradient threshold may not generalize; cleaning step is heuristic; scatter plot rendering may be slow for very large datasets.
- **When to use:** When visualizing sparse volumetric data or when noise reduction is required before plotting.
- **When not to use:** When continuous volume rendering is needed; when performance is critical for large datasets.

## Background
This method is derived from implementations in the Plenoxels repository [S1, unofficial] and GPUGems Volume Rendering examples [S2, unofficial]. It represents a simple approach to volume visualization by converting voxel data into a point cloud.

## Sources
[S1] DanJbk/Plenoxels, https://github.com/DanJbk/Plenoxels, fetched 2026-10-27 (unofficial)
[S2] sho3la/GPUGemsVolumeRendering, https://github.com/sho3la/GPUGemsVolumeRendering, fetched 2026-10-26 (unofficial)
