# Grid Visualization with Thresholding

Also called: 3D Scatter Plot Visualization, Thresholded Grid Visualization

## Overview
Grid Visualization with Thresholding is a sparse visualization technique that filters a 3D grid tensor based on opacity and gradient magnitude thresholds, then renders the surviving cells as a 3D scatter plot [S1, unofficial]. It solves the problem of visualizing volumetric data without the computational overhead of ray casting or volume integration, making it suitable for rapid inspection of sparse structures [S1, unofficial]. It sets itself apart from its sibling [3D Table Computation on GPU](2083-3d-table-computation-on-gpu.md) by avoiding precomputed lookup tables and instead performing direct scalar filtering and point plotting, and from its parent [2D Visualization from 3D Grid (Slow)](1507-2d-visualization-from-3d-grid-slow.md) by explicitly leveraging gradient magnitude for structural filtering and outputting a 3D point cloud rather than 2D slices [S1, unofficial].

## How it works
**Inputs:**
- `grid_cells`: A tensor of shape $(D, H, W, C)$ containing density/opacity values and RGB color channels, where $C \ge 1$.
- `gradients`: (Optional) A tensor of shape $(D, H, W, 3)$ containing the gradient vector for each cell.
- `threshold`: Float. Opacity threshold for visualization. Default: $0.0$ [S1, unofficial].
- `do_threshold`: Boolean. Whether to apply the opacity threshold. Default: `True` [S1, unofficial].
- `clean_strays`: Boolean. Whether to apply noise cleaning. Default: `False` [S1, unofficial].

**Outputs:**
- A list of 3D coordinates $(x, y, z)$ and corresponding RGB colors $(r, g, b)$ for all cells where the final alpha value is greater than $0$.

**Data Structures:**
- The grid is stored as a dense tensor. No explicit sparse structures (like linked lists or hash maps) are used in the core algorithm; sparsity is achieved by filtering.

**Steps:**
1. **Load Grid:** Load the `grid_cells` tensor onto the target device (e.g., CPU or GPU) [S1, unofficial].
2. **Gradient Filtering (if applicable):** If gradient data exists and thresholding is enabled, compute the gradient norm for each cell. Zero out cells where the gradient norm is below $3.0 \times 10^{-6}$ [S1, unofficial].
   - Formula: If $||\nabla f_{i,j,k}|| < 3.0 \times 10^{-6}$, then $grid\_cells_{i,j,k} = 0$.
3. **Clipping:** Clip all values in `grid_cells` to the range $[0, 1]$ [S1, unofficial].
4. **Opacity Thresholding:** If `do_threshold` is `True`:
   - Zero out alpha values (the first channel or the density channel) below the `threshold`.
   - Set all non-zero alpha values to $1.0$ [S1, unofficial].
   - Formula: If $alpha_{i,j,k} < threshold$, then $alpha_{i,j,k} = 0$. Else, $alpha_{i,j,k} = 1$.
5. **Noise Cleaning (if applicable):** If `clean_strays` is `True`, apply a convolution-based noise removal process twice with different parameters [S1, unofficial]:
   - First pass: Convolve with a kernel of size $3$ and radius $1$.
   - Second pass: Convolve with a kernel of size $7$ and radius $3$.
   - Note: The specific convolution implementation (e.g., mean filter, Gaussian) is not detailed in the source, but the sequence is fixed [S1, unofficial].
6. **Extraction:** Extract coordinates $(x, y, z)$ and RGB values for all cells where the final alpha value is $> 0$ [S1, unofficial].
7. **Plotting:** Render the extracted points using a 3D plotting library such as Plotly or Matplotlib [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def grid_visualization_with_thresholding(grid_cells, gradients=None, threshold=0.0, do_threshold=True, clean_strays=False):
    """
    Visualizes grid cells by filtering based on opacity threshold and gradient magnitude,
    then plotting as a 3D scatter plot.
    
    Args:
        grid_cells (torch.Tensor): Tensor of shape (D, H, W, C) with density/opacity in the first channel.
        gradients (torch.Tensor, optional): Tensor of shape (D, H, W, 3) with gradient vectors.
        threshold (float): Opacity threshold for visualization.
        do_threshold (bool): Whether to apply thresholding.
        clean_strays (bool): Whether to remove stray voxels.
        
    Returns:
        list: List of tuples (x, y, z, r, g, b) for visualization.
    """
    # 1. Load grid cells tensor (assumed already on device)
    # grid_cells is expected to be a torch.Tensor
    
    # 2. Gradient filtering
    if gradients is not None and do_threshold:
        # Compute gradient norm
        grad_norm = torch.norm(gradients, dim=-1) # Shape: (D, H, W)
        # Zero out cells where gradient norm is below 3.0e-06
        mask = grad_norm < 3.0e-06
        grid_cells[mask] = 0
        
    # 3. Clip values to [0, 1]
    grid_cells = torch.clamp(grid_cells, 0.0, 1.0)
    
    # 4. Opacity thresholding
    if do_threshold:
        # Assuming the first channel is opacity/density
        opacity = grid_cells[..., 0]
        # Zero out alpha values below threshold
        opacity[opacity < threshold] = 0.0
        # Set non-zero alphas to 1
        opacity[opacity > 0] = 1.0
        # Update grid_cells
        grid_cells[..., 0] = opacity
        
    # 5. Noise cleaning
    if clean_strays:
        # Apply convolve_grid_to_remove_noise twice
        # Note: The specific convolution function is not provided in the source.
        # We assume a placeholder for the convolution operation.
        # Pass 1: kernel 3, radius 1
        grid_cells = convolve_grid(grid_cells, kernel_size=3, radius=1)
        # Pass 2: kernel 7, radius 3
        grid_cells = convolve_grid(grid_cells, kernel_size=7, radius=3)
        
    # 6. Extract coordinates and RGB values for cells with alpha > 0
    # Assuming alpha is the first channel
    alpha = grid_cells[..., 0]
    valid_mask = alpha > 0
    
    # Get indices of valid cells
    indices = torch.nonzero(valid_mask, as_tuple=False)
    
    # Extract coordinates (z, y, x) from indices
    # Note: Plotly/Matplotlib usually expect (x, y, z)
    x = indices[:, 2].float()
    y = indices[:, 1].float()
    z = indices[:, 0].float()
    
    # Extract RGB values (assuming channels 1, 2, 3 are RGB)
    # If C < 4, handle appropriately. Assuming C >= 4 for RGB.
    if grid_cells.shape[-1] >= 4:
        r = grid_cells[valid_mask, 1]
        g = grid_cells[valid_mask, 2]
        b = grid_cells[valid_mask, 3]
    else:
        # Fallback if RGB is not present, use grayscale or default
        r = grid_cells[valid_mask, 0]
        g = grid_cells[valid_mask, 0]
        b = grid_cells[valid_mask, 0]
        
    # 7. Plot using Plotly or Matplotlib
    # This step is external; return data for plotting
    points = []
    for i in range(len(x)):
        points.append((x[i].item(), y[i].item(), z[i].item(), r[i].item(), g[i].item(), b[i].item()))
        
    return points

def convolve_grid(tensor, kernel_size, radius):
    """
    Placeholder for the convolution operation described in the source.
    The source mentions 'convolve_grid_to_remove_noise' with specific kernel/radius parameters.
    The exact implementation is not provided in the source.
    """
    # Not given in the sources.
    return tensor
```

**Parameters:**
- `threshold`: Opacity threshold for visualization. Default: $0.0$ [S1, unofficial].
- `do_threshold`: Whether to apply thresholding. Default: `True` [S1, unofficial].
- `clean_strays`: Whether to remove stray voxels. Default: `False` [S1, unofficial].

**Pitfalls:**
- Gradient-based thresholding uses a fixed threshold of $3.0 \times 10^{-6}$ for gradient norm, which may not be optimal for all datasets [S1, unofficial].
- Visualization thresholding uses a configurable opacity threshold, which is distinct from the gradient threshold [S1, unofficial].
- The noise cleaning step relies on an unspecified convolution function, which may lead to implementation gaps if the exact kernel is not defined [S1, unofficial].

## Subtopics

### [Density Range Windowing](1571-density-range-windowing.md)
Density Range Windowing restricts rendering to voxels whose density falls within a specified $[min, max]$ range, allowing selective visualization of specific structures within a volume. It sets itself apart from its parent method, Grid Visualization with Thresholding, by operating on a continuous density interval rather than a single binary opacity threshold or gradient magnitude, enabling finer control over semi-transparent vs. dense core separation.

### [Glyph Lines](8934-glyph-lines.md)
Glyph Lines is a sparse visualization technique for 3D vector fields that samples the field on a 2D grid at a fixed coordinate and renders each sample as a line segment. It solves the problem of visualizing flow direction and magnitude in a specific slice of a volumetric dataset without the computational cost of full volume ray casting. It differs from its parent method, Grid Visualization with Thresholding, by explicitly encoding vector direction and magnitude into geometric primitives (lines) rather than plotting scalar grid cells as points.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels, as it involves a single pass over the grid for filtering and extraction.
- **Memory Complexity:** $O(N)$ for storing the grid and the resulting point cloud.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of voxels. Performance is limited by the extraction and plotting steps for very large grids.
- **Strengths:** Simple to implement; fast for sparse data; no need for complex ray casting or volume integration.
- **Limitations:** Loses depth information and occlusion handling; not suitable for dense volumes; relies on external plotting libraries for visualization.
- **When to use:** Rapid inspection of sparse volumetric data; debugging; educational purposes.
- **When not to use:** Dense volumes; production-quality rendering; when occlusion or depth cues are critical.

## Background
Origin: The method is described in the unofficial source [S1, unofficial], which appears to be a GitHub repository (DanJbk/Plenoxels). The specific origin date and author are not provided in the source.

## Sources
[S1] DanJbk/Plenoxels, https://github.com/DanJbk/Plenoxels, fetched 2026-10-27 (unofficial)
