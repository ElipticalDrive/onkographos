# Automated Classification Widget Placement

Also called: Automated Widget Placement

## Overview
Automated Classification Widget Placement is an interaction technique that automatically positions and sizes classification widgets in the transfer-function domain based on data probe feedback and reprojected voxel distributions [S1, book][S2]. It solves the difficulty of manually creating effective higher-dimensional transfer functions by tying widget geometry to the statistical distribution of voxels selected by a probe [S1, book][S2]. Unlike manual editing, this method reduces user effort by deriving widget parameters directly from the data's local characteristics at the probe location [S1, book][S2].

## How it works
**Inputs and Outputs**
*   **Input:** A data probe object with a tip position, a slider value, and a query location. A set of reprojected voxels associated with the probe's current selection or neighborhood.
*   **Output:** The position and dimensions (size) of classification widgets (rectangular or triangular) in the transfer-function domain.

**Data Structures**
*   **Data Probe:** Contains a tip coordinate (in transfer-function space), a slider value (scalar), and a query data value.
*   **Reprojected Voxels:** A collection of voxel samples mapped to the transfer-function domain, used to calculate statistical variance.

**Steps**
1.  **Retrieve Probe State:** Get the transfer-function values interpolated at the data probe’s tip and the current slider value [S1, book][S2].
2.  **Query Data Value:** Determine the specific data value queried by the probe [S1, book][S2].
3.  **Calculate Variance:** Compute the horizontal variance and vertical variance of the reprojected voxel locations relative to the probe's query point [S1, book][S2].
4.  **Place Rectangular Widget:**
    *   Center the widget at the transfer-function values interpolated at the data probe’s tip [S1, book][S2].
    *   Set the size (dimensions) of the rectangle based on the data probe’s slider value [S1, book][S2].
5.  **Place Triangular Widget:**
    *   Locate the widget horizontally at the data value queried by the probe [S1, book][S2].
    *   Determine the width of the triangle using the horizontal variance of the reprojected voxel locations [S1, book][S2].
    *   Determine the height of the triangle using the vertical variance of the reprojected voxel locations [S1, book][S2].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def place_widgets(probe, reprojected_voxels):
    """
    Places rectangular and triangular classification widgets based on probe feedback.
    
    Args:
        probe: Object containing tip_tf_values (vector), slider_value (float), 
               and query_data_value (float).
        reprojected_voxels: List of voxel positions in transfer-function space.
        
    Returns:
        rect_widget: Dictionary with 'center' and 'size'.
        tri_widget: Dictionary with 'x_position', 'width', and 'height'.
    """
    # Step 1 & 2: Retrieve probe state
    tip_tf_values = probe.tip_tf_values
    slider_value = probe.slider_value
    query_data_value = probe.query_data_value
    
    # Step 3: Calculate variance of reprojected voxels
    # Assuming voxels are points (x, y) in TF space
    if len(reprojected_voxels) == 0:
        # [G] Handle empty voxel set to avoid division by zero or undefined variance
        horizontal_variance = 0.0
        vertical_variance = 0.0
    else:
        # Calculate mean
        mean_x = sum(v.x for v in reprojected_voxels) / len(reprojected_voxels)
        mean_y = sum(v.y for v in reprojected_voxels) / len(reprojected_voxels)
        
        # Calculate variance
        horizontal_variance = sum((v.x - mean_x)**2 for v in reprojected_voxels) / len(reprojected_voxels)
        vertical_variance = sum((v.y - mean_y)**2 for v in reprojected_voxels) / len(reprojected_voxels)
        
    # Step 4: Place Rectangular Widget
    # Centered at tip values, size controlled by slider
    rect_widget = {
        'center': tip_tf_values,
        'size': slider_value  # [G] Specific mapping of slider to size (e.g., width/height) is not detailed in sources
    }
    
    # Step 5: Place Triangular Widget
    # Horizontal position at query data value
    # Width and height determined by horizontal and vertical variance
    tri_widget = {
        'x_position': query_data_value,
        'width': horizontal_variance,
        'height': vertical_variance
    }
    
    return rect_widget, tri_widget
```

**Parameters and Edge Cases**
*   **Slider Value:** Controls the size of the rectangular widget. The exact scaling factor from slider value to widget dimensions is not specified [S1, book][S2].
*   **Variance Calculation:** The sources specify "horizontal and vertical variance" but do not define the reference point for the variance calculation (e.g., mean of all voxels vs. mean relative to the probe tip) [S1, book][S2]. The pseudocode assumes global mean of the reprojected voxels [G].
*   **Empty Voxels:** If no voxels are reprojected, variance is undefined. A guard clause is required [G].

## Subtopics

### Position-dependent Transfer Function Customization
Allows customization based on spatial position, differing from automated placement which relies on statistical variance of reprojected voxels [5287-position-dependent-transfer-function-customization.md].

### Sketch-based Transfer Function Editing
Relies on user-drawn sketches rather than automated statistical derivation from probe feedback [5943-sketch-based-transfer-function-editing.md].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of reprojected voxels for variance calculation [S1, book][S2].
*   **Memory Complexity:** Proportional to the number of reprojected voxels stored for variance computation [S1, book][S2].
*   **Strengths:** Reduces the difficulty of creating effective higher-dimensional transfer functions by automating widget placement [S1, book][S2].
*   **Limitations:** Relies on the accuracy of the reprojected voxel distribution; if the probe selection is poor, the automated placement may be misleading [S1, book][S2].
*   **When to use:** When users need to quickly adjust transfer functions in multi-dimensional spaces without manual tuning of widget boundaries [S1, book][S2].
*   **When not to use:** When precise, artist-driven control over widget boundaries is required, or when the voxel distribution is sparse or noisy [S1, book][S2].

## Background
This method was described in the context of multi-dimensional transfer functions for volume rendering [S1, book][S2]. It is part of a broader set of interaction techniques including shading and color-picker widgets [S1, book][S2]. The technique was presented in works by Kniss et al. around 2005 [S1, book][S2].

## Sources
[S1] PDF 9 Multi-Dimensional Transfer Functions for Volume Rendering, https://people.cs.uchicago.edu/~glk/pubs/pdf/Kniss-MultidimensionalTransferFunctions-VH-2005.pdf, fetched 2026-10-27 (book)
[S2] PDF Interactive Volume Rendering Using Multi-Dimensional Transfer Functions ..., https://collections.lib.utah.edu/dl_files/d4/e4/d4e47bfc09e54eee9f532e4f22b17e9214f4c755.pdf, fetched 2026-10-27
