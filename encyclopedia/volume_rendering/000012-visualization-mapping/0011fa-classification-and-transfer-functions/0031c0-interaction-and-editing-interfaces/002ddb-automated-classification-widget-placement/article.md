# Automated Classification Widget Placement

Also called: Automated Widget Placement

## Overview
Automated Classification Widget Placement is an interaction technique that positions and sizes classification widgets in the transfer-function domain based on data probe feedback and reprojected voxel distributions [S1, book][S2]. It reduces the difficulty of manually creating effective higher-dimensional transfer functions by tying widget geometry to the statistical distribution of data at a specific probe location [S1, book][S2]. The method builds on [Interaction and Editing Interfaces](6592-interaction-and-editing-interfaces.md).

## How it works
**Inputs and Outputs:**
*   **Input:** A data probe defined by its tip position and slider value, and a set of voxels reprojected to the probe's current query state [S1, book][S2].
*   **Output:** The position and dimensions of a rectangular or triangular classification widget in the transfer-function domain [S1, book][S2].

**Data Structures:**
The method operates on a probe state containing the tip coordinates, slider value, and a list of reprojected voxel coordinates mapped to the transfer-function domain [S1, book][S2].

**Steps:**
1. Query the transfer-function values interpolated at the data probe’s tip [S1, book][S2].
2. Identify the set of voxels reprojected to the current probe state [S1, book][S2].
3. Compute the horizontal and vertical variance of the reprojected voxel locations in the transfer-function domain [S1, book][S2].
4. Place the widget:
   * **Rectangular Widget:** Center it at the transfer-function values interpolated at the data probe’s tip [S1, book][S2]. Control its size using the data probe’s slider [S1, book][S2].
   * **Triangular Widget:** Locate it horizontally at the data value queried by the probe [S1, book][S2]. Determine its width and height from the horizontal and vertical variance in the reprojected voxel locations [S1, book][S2].

**Formulas:**
The exact mathematical mapping from the slider value to widget dimensions, and from voxel variance to widget width/height, is Not given in the sources [S1, book][S2]. Standard variance calculation applies to the reprojected voxel coordinates [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def place_widget(probe_tip_values, reprojected_voxels, slider_value, widget_type):
    # probe_tip_values: tuple of transfer function values at probe tip
    # reprojected_voxels: list of tuples (x, y) representing voxel positions in TF domain
    # slider_value: scalar controlling widget size
    # widget_type: "rectangular" or "triangular"

    if not reprojected_voxels:
        return None  # Fallback behavior not given in the sources [S1, book][S2].

    # Calculate horizontal and vertical variance of reprojected voxels [G]
    xs = [v[0] for v in reprojected_voxels]
    ys = [v[1] for v in reprojected_voxels]
    mean_x = sum(xs) / len(xs)
    mean_y = sum(ys) / len(ys)
    var_h = sum((x - mean_x)**2 for x in xs) / len(xs)
    var_v = sum((y - mean_y)**2 for y in ys) / len(ys)

    if widget_type == "rectangular":
        # Center at interpolated transfer function values at probe tip
        center = probe_tip_values
        # Size controlled by slider (exact mapping not given in the sources [S1, book][S2])
        width = slider_value
        height = slider_value
        return {"type": "rectangular", "center": center, "size": (width, height)}

    elif widget_type == "triangular":
        # Horizontal position at data value queried by probe
        x_pos = probe_tip_values[0]
        # Width and height determined by horizontal and vertical variance
        # Scaling factors from variance to dimensions are not given in the sources [S1, book][S2]
        width = var_h
        height = var_v
        return {"type": "triangular", "position": (x_pos, 0), "dimensions": (width, height)}
```

**Edge Cases:**
*   **Empty Probe:** If no voxels are reprojected, variance cannot be computed [S1, book][S2]. The sources do not specify a fallback behavior [S1, book][S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, book][S2].
*   **Memory Complexity:** Not given in the sources [S1, book][S2].
*   **Strengths:** Reduces the difficulty of creating effective higher-dimensional transfer functions by automating widget placement based on probe feedback [S1, book][S2].
*   **Limitations:** Not given in the sources [S1, book][S2].
*   **When to Use:** Not given in the sources [S1, book][S2].
*   **When Not to Use:** Not given in the sources [S1, book][S2].

## Background
The method was introduced in the context of multi-dimensional transfer functions for volume rendering [S1, book]. It addresses the complexity of higher-dimensional transfer functions by automating widget placement based on probe feedback [S1, book][S2]. The technique is part of a broader set of interaction widgets including shading and color-picker widgets [S1, book][S2].

## Sources
[S1] PDF 9 Multi-Dimensional Transfer Functions for Volume Rendering, https://people.cs.uchicago.edu/~glk/pubs/pdf/Kniss-MultidimensionalTransferFunctions-VH-2005.pdf, fetched 2026-10-27 (book)
[S2] PDF Interactive Volume Rendering Using Multi-Dimensional Transfer Functions ..., https://collections.lib.utah.edu/dl_files/d4/e4/d4e47bfc09e54eee9f532e4f22b17e9214f4c755.pdf, fetched 2026-10-27
