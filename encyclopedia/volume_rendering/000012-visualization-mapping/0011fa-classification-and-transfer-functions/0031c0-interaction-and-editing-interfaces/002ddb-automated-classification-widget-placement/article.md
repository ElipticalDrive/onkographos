# Automated Classification Widget Placement

Also called: Automated Widget Placement

## Overview
Automated Classification Widget Placement is an interaction technique for multi-dimensional transfer functions that automatically positions and sizes classification widgets based on data probe feedback and the statistical distribution of reprojected voxels [S1, book][S2]. It solves the difficulty of manually configuring effective higher-dimensional transfer functions by tying widget geometry directly to the local data distribution queried by the user [S1, book][S2]. Unlike manual editing interfaces, it reduces the cognitive load of widget placement by deriving location and scale from probe state and voxel variance [S1, book][S2].

## How it works
**Inputs and Outputs**
- **Input:** A data probe object containing a tip position (data value), a slider value (control parameter), and a set of reprojected voxel locations corresponding to the probe's current query region.
- **Output:** Updated geometry (position and dimensions) for rectangular and triangular classification widgets in the transfer-function domain.

**Data Structures**
- **Data Probe:** An interactive handle in the transfer-function domain.
  - `tip_value`: The scalar data value at the probe's tip.
  - `slider_value`: A scalar controlling the scale of the widget.
- **Reprojected Voxels:** A list of voxel coordinates $(x_i, y_i)$ in the transfer-function domain that map to the probe's current selection.

**Steps**
1. **Query Voxels:** Identify the set of voxels reprojected to the transfer-function domain based on the current data probe location. Let this set be $V = \{(x_i, y_i) | i = 1 \dots N\}$.
2. **Calculate Statistics:** Compute the horizontal variance ($\sigma_x^2$) and vertical variance ($\sigma_y^2$) of the reprojected voxel locations.
   $$ \sigma_x^2 = \frac{1}{N} \sum_{i=1}^{N} (x_i - \bar{x})^2 $$
   $$ \sigma_y^2 = \frac{1}{N} \sum_{i=1}^{N} (y_i - \bar{y})^2 $$
   Where $\bar{x}$ and $\bar{y}$ are the mean coordinates of the voxels in $V$.
3. **Position Rectangular Widget:**
   - Center the rectangular widget at the transfer-function values interpolated at the data probe’s tip.
   - Set the size of the rectangle using the data probe’s slider value.
4. **Position Triangular Widget:**
   - Locate the triangular widget horizontally at the data value queried by the probe (the probe's tip value).
   - Determine the width of the triangle using the horizontal variance ($\sigma_x$) of the reprojected voxels.
   - Determine the height of the triangle using the vertical variance ($\sigma_y$) of the reprojected voxels.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_widgets(probe, reprojected_voxels):
    """
    probe: object with attributes .tip_value (float) and .slider_value (float)
    reprojected_voxels: list of tuples (x, y) representing voxel positions in TF domain
    """
    if not reprojected_voxels:
        return

    # Calculate mean and variance of reprojected voxels
    n = len(reprojected_voxels)
    xs = [v[0] for v in reprojected_voxels]
    ys = [v[1] for v in reprojected_voxels]
    
    mean_x = sum(xs) / n
    mean_y = sum(ys) / n
    
    var_x = sum((x - mean_x)**2 for x in xs) / n
    var_y = sum((y - mean_y)**2 for y in ys) / n
    
    # Standard deviations for width/height determination
    std_x = var_x ** 0.5
    std_y = var_y ** 0.5

    # Update Rectangular Widget
    # Centered at probe tip values
    rect_center_x = probe.tip_value
    rect_center_y = probe.tip_value # Assuming isotropic interpolation or specific TF dimension mapping
    # Size controlled by slider
    rect_width = probe.slider_value
    rect_height = probe.slider_value
    
    # Update Triangular Widget
    # Located horizontally at data value queried by probe
    tri_base_x = probe.tip_value
    # Width determined by horizontal variance
    tri_width = std_x
    # Height determined by vertical variance
    tri_height = std_y
    
    return {
        'rect': {'center': (rect_center_x, rect_center_y), 'size': (rect_width, rect_height)},
        'tri': {'base_x': tri_base_x, 'width': tri_width, 'height': tri_height}
    }
```

**Parameters**
- **Slider Value:** Controls the scale of the rectangular widget. Typical values depend on the transfer function domain normalization (e.g., 0.0 to 1.0) [S1, book][S2].
- **Variance Scaling:** The sources do not specify a scaling factor for the variances when determining triangle dimensions. It is assumed the variance is used directly or requires normalization based on the specific transfer function domain range [S1, book][S2].

**Edge Cases**
- **Empty Voxels:** If the probe selects no voxels, variance is undefined. The implementation must handle this case, likely by defaulting to a minimal widget size or ignoring the update [G].
- **Zero Variance:** If all reprojected voxels are at the same location, variance is zero, resulting in a degenerate widget (zero width/height). A minimum threshold may be required [G].

## Subtopics

### Rectangular Widget Placement
Centers the widget at the probe tip and uses the slider for size. Choose when a fixed-size region around the probe tip is desired [S1, book][S2].

### Triangular Widget Placement
Uses variance of reprojected voxels for width and height. Choose when the widget should adapt to the spread of data in the transfer-function domain [S1, book][S2].

## Operating characteristics
- **Time Complexity:** $O(N)$ per update, where $N$ is the number of reprojected voxels, due to the calculation of mean and variance [S1, book][S2].
- **Memory Complexity:** $O(N)$ to store the reprojected voxel coordinates [S1, book][S2].
- **Strengths:** Reduces the difficulty of creating effective higher-dimensional transfer functions by automating placement based on data feedback [S1, book][S2].
- **Limitations:** Relies on the accuracy of the reprojected voxel distribution; if the probe query is ambiguous or the data distribution is noisy, the widget placement may be unstable [S1, book][S2].
- **When to Use:** When users need to quickly adapt transfer function regions to local data characteristics without manual tuning [S1, book][S2].
- **When Not to Use:** When precise, manual control over widget boundaries is required for specific artistic or analytical goals [S1, book][S2].

## Background
This method was described by Kniss et al. in the context of multi-dimensional transfer functions for volume rendering [S1, book][S2]. It is part of a broader set of interaction techniques including shading and color-picker widgets, aimed at simplifying the complexity of higher-dimensional transfer function design [S1, book][S2].

## Sources
[S1] PDF 9 Multi-Dimensional Transfer Functions for Volume Rendering, https://people.cs.uchicago.edu/~glk/pubs/pdf/Kniss-MultidimensionalTransferFunctions-VH-2005.pdf, fetched 2026-10-27 (book)
[S2] PDF Interactive Volume Rendering Using Multi-Dimensional Transfer Functions ..., https://collections.lib.utah.edu/dl_files/d4/e4/d4e47bfc09e54eee9f532e4f22b17e9214f4c755.pdf, fetched 2026-10-27
