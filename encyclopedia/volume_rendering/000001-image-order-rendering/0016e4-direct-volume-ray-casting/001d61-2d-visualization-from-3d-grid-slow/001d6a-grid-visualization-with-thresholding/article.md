# Grid Visualization with Thresholding

Also called: 3D Scatter Plot Visualization, Thresholded Grid Visualization

## Overview
Grid Visualization with Thresholding filters a 3D scalar grid based on opacity and gradient magnitude thresholds, then renders the surviving cells as a 3D scatter plot [S1, unofficial]. It solves the problem of visualizing sparse volumetric structures without the computational overhead of full volume ray casting [S1, unofficial]. It differs from its parent method, [2D Visualization from 3D Grid (Slow)](1507-2d-visualization-from-3d-grid-slow.md), by explicitly encoding scalar values into point coordinates and colors rather than projecting slices, and from [3D Table Computation on GPU](2083-3d-table-computation-on-gpu.md) by operating in CPU-space Python libraries (Plotly/Matplotlib) rather than GPU shader pipelines [S1, unofficial].

## How it works
**Inputs:** A 3D grid tensor of scalar values (typically normalized to $[0, 1]$), optional gradient data tensor, and boolean flags for thresholding and cleaning.
**Outputs:** A 3D scatter plot visualization where each point represents a grid cell with coordinates $(x, y, z)$ and color $(R, G, B)$ derived from the scalar value.

**Data Structures:**
- `grid_cells`: Tensor of shape $(D, H, W)$ containing scalar values.
- `gradients`: Optional tensor of shape $(D, H, W, 3)$ containing gradient vectors $\nabla f$.
- `threshold`: Float, opacity threshold for visualization (default 0.0).
- `do_threshold`: Boolean, whether to apply thresholding (default True).
- `clean_strays`: Boolean, whether to remove stray voxels (default False).

**Steps:**
1. Load the grid cells tensor.
2. If gradient data exists and thresholding is enabled, compute the gradient norm $\|\nabla f\|$ for each cell. Zero out cells where $\|\nabla f\| < 3.0 \times 10^{-6}$ [S1, unofficial].
3. Clip all grid values to the range $[0, 1]$ [S1, unofficial].
4. If thresholding is enabled, zero out alpha values below the `threshold` parameter. Set all non-zero alpha values to 1.0 [S1, unofficial].
5. If `clean_strays` is enabled, apply noise removal by convolving the grid twice: first with a kernel of size 3 and radius 1, then with a kernel of size 7 and radius 3 [S1, unofficial].
6. Extract coordinates $(x, y, z)$ and RGB values for all cells where the final alpha value is greater than 0 [S1, unofficial].
7. Plot the extracted points using a 3D plotting library such as Plotly or Matplotlib [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch

def grid_visualization_with_thresholding(grid_cells, gradients=None, threshold=0.0, do_threshold=True, clean_strays=False):
    """
    Visualizes grid cells by filtering based on opacity threshold and gradient magnitude,
    then plotting as a 3D scatter plot.
    
    Args:
        grid_cells: Tensor of shape (D, H, W) with scalar values.
        gradients: Optional Tensor of shape (D, H, W, 3) with gradient vectors.
        threshold: Opacity threshold for visualization. Default is 0.0.
        do_threshold: Whether to apply thresholding. Default is True.
        clean_strays: Whether to remove stray voxels. Default is False.
        
    Returns:
        x, y, z, r, g, b: Arrays of coordinates and colors for plotting.
    """
    # Step 1: Load grid cells tensor (already loaded as input)
    # Ensure grid is on CPU for numpy operations if needed, or keep on device
    grid = grid_cells.clone()
    
    # Step 2: Gradient-based filtering
    if gradients is not None and do_threshold:
        # Compute gradient norm
        grad_norm = torch.norm(gradients, dim=-1) # Shape (D, H, W)
        # Zero out cells where gradient norm is below 3.0e-06
        grid[grad_norm < 3.0e-06] = 0.0
        
    # Step 3: Clip values to [0, 1]
    grid = torch.clamp(grid, 0.0, 1.0)
    
    # Step 4: Opacity thresholding
    if do_threshold:
        # Create alpha mask: 1.0 if grid > threshold, else 0.0
        alpha = (grid > threshold).float()
        # Zero out alpha values below threshold
        grid = grid * alpha
        # Set non-zero alphas to 1 (for visualization purposes, we keep the scalar value for color)
        # Note: The source says "set non-zero alphas to 1", implying the alpha channel is binary
        # but the scalar value is preserved for RGB mapping.
        # We will use the original grid value for color, but mask out invalid cells.
        valid_mask = (grid > 0)
        
    else:
        valid_mask = (grid > 0)
        
    # Step 5: Cleaning strays
    if clean_strays:
        # Apply convolve_grid_to_remove_noise twice
        # Kernel 3/radius 1
        grid = convolve_grid_to_remove_noise(grid, kernel_size=3, radius=1)
        # Kernel 7/radius 3
        grid = convolve_grid_to_remove_noise(grid, kernel_size=7, radius=3)
        # Re-apply valid mask after cleaning if necessary, or re-evaluate threshold
        if do_threshold:
            valid_mask = (grid > 0)
            
    # Step 6: Extract coordinates and RGB values
    # Get indices of valid cells
    indices = torch.nonzero(valid_mask, as_tuple=True)
    x = indices[0].float()
    y = indices[1].float()
    z = indices[2].float()
    
    # Get scalar values for color mapping
    values = grid[valid_mask]
    
    # Map scalar values to RGB (simple grayscale or jet colormap implementation)
    # For simplicity, we map value to RGB directly (grayscale)
    r = values
    g = values
    b = values
    
    # Step 7: Plot using Plotly or Matplotlib
    # This step is external to the data extraction logic
    return x.numpy(), y.numpy(), z.numpy(), r.numpy(), g.numpy(), b.numpy()

def convolve_grid_to_remove_noise(grid, kernel_size, radius):
    """
    Simple noise removal by convolving with a box filter.
    """
    # Pad the grid
    pad = radius
    padded_grid = torch.nn.functional.pad(grid, (pad, pad, pad, pad, pad, pad), mode='constant', value=0)
    
    # Create a box kernel
    kernel = torch.ones((1, 1, kernel_size, kernel_size, kernel_size))
    
    # Convolve
    # Note: This is a simplified implementation. Real implementation might use scipy or torch.nn.functional.conv3d
    # Assuming grid is (D, H, W), we need to add channel dimension for conv3d
    if grid.dim() == 3:
        grid = grid.unsqueeze(0).unsqueeze(0)
        
    convolved = torch.nn.functional.conv3d(grid, kernel, padding=radius)
    
    # Normalize by kernel volume
    convolved = convolved / (kernel_size ** 3)
    
    # Remove padding and channel dimensions
    convolved = convolved.squeeze(0).squeeze(0)
    
    return convolved
```

**Parameters:**
- `threshold`: Opacity threshold for visualization. Default is 0.0 [S1, unofficial].
- `do_threshold`: Whether to apply thresholding. Default is True [S1, unofficial].
- `clean_strays`: Whether to remove stray voxels. Default is False [S1, unofficial].

**Pitfalls:**
- Gradient-based thresholding uses a fixed threshold of $3.0 \times 10^{-6}$ for gradient norm, which may not be suitable for all datasets [S1, unofficial].
- Visualization thresholding uses a configurable opacity threshold, which must be tuned per dataset [S1, unofficial].
- The cleaning step assumes a specific noise structure; inappropriate kernel sizes may blur important features [S1, unofficial].

## Subtopics

### [Glyph Lines](8934-glyph-lines.md)
Glyph Lines is a sparse visualization technique for 3D vector fields that samples the field on a 2D grid at a fixed coordinate and renders each sample as a line segment. It solves the problem of visualizing flow direction and magnitude in a specific slice of a volumetric dataset without the computational cost of full volume ray casting. It differs from its parent method, Grid Visualization with Thresholding, by explicitly encoding vector direction and magnitude into geometric primitives (lines) rather than plotting scalar grid cells as points.

### Density Range Windowing
Density Range Windowing restricts rendering to voxels whose density falls within a specified $[min, max]$ range, allowing selective visualization of structures [S2, unofficial]. It raises `min` to drop semi-transparent structures (skin/foliage) and retain dense cores (bone), and lowers `max` to peel off densest structures [S2, unofficial]. Parameters `m_densityMin` (default 0.0f) and `m_densityMax` (default 1.0f) define the visible density interval [S2, unofficial].

## Operating characteristics
**Time Complexity:** $O(N)$ where $N$ is the number of grid cells, for filtering and extraction. Plotting time depends on the number of visible points and the rendering engine.
**Memory Complexity:** $O(N)$ to store the grid and intermediate tensors.
**Measured Performance:** Not given in the sources.
**Scaling:** Performance degrades linearly with grid size. Plotting libraries may struggle with very large point clouds.
**Strengths:** Simple to implement, fast for sparse data, easy to integrate with standard Python visualization tools.
**Limitations:** Not suitable for dense volumetric data, lacks depth cues and occlusion handling, limited to scalar fields (unless extended).
**When to use:** Quick inspection of sparse volumetric data, debugging, or when GPU-based rendering is not available.
**When not to use:** Dense volumetric visualization, real-time applications, or when accurate depth perception is required.

## Background
Grid Visualization with Thresholding originates from practical implementations in Python-based volumetric data analysis, specifically within the Plenoxels project [S1, unofficial]. It is used for rapid visualization of 3D scalar fields, particularly in contexts where full volume ray casting is overkill or unavailable. The method leverages standard scientific computing libraries (NumPy, PyTorch) and visualization tools (Plotly, Matplotlib) to provide an accessible entry point for 3D data exploration.

## Sources
[S1] DanJbk/Plenoxels, https://github.com/DanJbk/Plenoxels, fetched 2026-10-27 (unofficial)
[S2] sho3la/GPUGemsVolumeRendering, https://github.com/sho3la/GPUGemsVolumeRendering, fetched 2026-10-26 (unofficial)
