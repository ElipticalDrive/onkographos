# 2D Transfer Function Widget

Also called: 2D TF Display, TF Widget, Transfer Function Widget

## Overview
The 2D Transfer Function Widget is an interactive visualization component that presents a 2D slice of the 3D transfer function (TF) domain, allowing users to manually set and alter the transfer function while receiving immediate feedback from volume queries [S1]. It distinguishes itself from standard piecewise linear editors by integrating "re-projected voxel" visualization, which maps spatial voxel structures directly into the TF domain to help users interpret local volume characteristics like gradient and curvature [S1]. This method bridges the gap between data space and TF space, enabling users to see how specific spatial features (e.g., boundaries vs. homogeneous regions) map to TF variables [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset; a query point in the volume (provided by a Data Probe or Clipping Plane widget); the current 3D Lookup Table (LUT) defining the transfer function.
*   **Output:** A 2D graphical widget displaying the TF domain slice, colored balls representing data values and derivatives, and edges representing voxel connectivity.

**Data Structures and Layout:**
The widget is implemented as a basic frame widget [S1].
*   **Horizontal Axis:** Data value ($v$).
*   **Vertical Axis:** Gradient magnitude ($|\nabla v|$).
*   **Third Axis:** Second derivative ($\nabla^2 v$) is not explicitly plotted as a coordinate but is encoded via color [S1].
*   **Inner Plane:** A polygon texture-mapped with a 2D slice through the 3D LUT, showing the full 3D transfer function context [S1].
*   **Controls:**
    *   Balls at the four corners of the widget frame allow resizing (scaling the TF domain range) [S1].
    *   Bars on the edges allow translation (shifting the TF domain range) [S1].

**Algorithm:**
1.  **Query Identification:** The system receives a query point $P$ from a volume query tool (Probe or Clip) [S1].
2.  **Voxel Localization:** Identify the voxel $V$ containing $P$ [S1].
3.  **Feature Extraction:** Calculate the data value $v(P)$ and its first derivatives $\nabla v(P)$ at $P$. Calculate the data values $v(C_i)$ and derivatives $\nabla v(C_i)$ for the eight corner samples $C_i$ of voxel $V$ [S1].
4.  **TF Domain Mapping:**
    *   Map $P$ to a point in the 2D TF plane: $(x, y) = (v(P), |\nabla v(P)|)$ [S1].
    *   Map each corner $C_i$ to a point in the 2D TF plane: $(x_i, y_i) = (v(C_i), |\nabla v(C_i)|)$ [S1].
5.  **Color Encoding:** Determine the second derivative $\nabla^2 v$ for the query point (and potentially corners, though sources emphasize the query point's color mapping) [S1].
    *   If $\nabla^2 v < 0$: Color is Blue.
    *   If $\nabla^2 v = 0$: Color is White.
    *   If $\nabla^2 v > 0$: Color is Yellow [S1].
6.  **Rendering:**
    *   Render a small ball at $(v(P), |\nabla v(P)|)$ colored by the second derivative rule [S1].
    *   Render eight balls at $(v(C_i), |\nabla v(C_i)|)$ [S1].
    *   Draw edges connecting the eight corner balls to reflect the spatial connectivity of the voxel corners (forming a cube-like structure in the TF domain) [S1].
7.  **Interaction:** The user observes the geometric configuration of the projected points. A cluster of points indicates a homogeneous region; a spread-out or distorted configuration indicates a boundary or complex structure [S1]. The user can then adjust the transfer function parameters (opacity/color) in the TF domain to affect the rendering of the queried region [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_tf_widget(query_point, volume_data, voxel_grid, lut_slice):
    """
    Renders the 2D Transfer Function Widget for a given query point.
    """
    # 1. Identify the voxel containing the query point
    voxel = voxel_grid.get_voxel_containing(query_point)
    
    # 2. Get the 8 corner coordinates of the voxel
    corner_coords = voxel.get_corner_coordinates()
    
    # 3. Calculate TF domain coordinates for the query point
    q_val = volume_data.get_value(query_point)
    q_grad_mag = volume_data.get_gradient_magnitude(query_point)
    q_second_deriv = volume_data.get_second_derivative(query_point)
    
    # Map to 2D TF plane
    q_tf_x = q_val
    q_tf_y = q_grad_mag
    
    # Determine color based on second derivative
    if q_second_deriv < 0:
        q_color = BLUE
    elif q_second_deriv == 0:
        q_color = WHITE
    else:
        q_color = YELLOW
        
    # 4. Calculate TF domain coordinates for the 8 corners
    corner_tf_points = []
    for corner in corner_coords:
        c_val = volume_data.get_value(corner)
        c_grad_mag = volume_data.get_gradient_magnitude(corner)
        
        # Map to 2D TF plane
        c_tf_x = c_val
        c_tf_y = c_grad_mag
        corner_tf_points.append((c_tf_x, c_tf_y))
        
    # 5. Render the widget
    # Draw the background LUT slice
    draw_texture(lut_slice)
    
    # Draw edges connecting the 8 corner balls (voxel connectivity)
    draw_edges(corner_tf_points, connectivity=VOXEL_CORNER_CONNECTIVITY)
    
    # Draw the 8 corner balls
    for pt in corner_tf_points:
        draw_ball(pt, color=GRAY) # Color of corners not specified, assume neutral
        
    # Draw the query point ball
    draw_ball((q_tf_x, q_tf_y), color=q_color)
    
    # Draw resize/translate controls
    draw_resize_balls(widget_bounds)
    draw_translate_bars(widget_bounds)
```

**Parameters:**
*   **TF Domain Range:** Defined by the widget frame. Resizable via corner balls [S1].
*   **Color Mapping:** Blue (negative 2nd deriv), White (zero), Yellow (positive) [S1].

**Edge Cases:**
*   **Precision:** Derivative calculations at voxel corners and query points must be consistent to ensure accurate mapping [S1].
*   **Degenerate Input:** If the query point lies exactly on a voxel boundary, the voxel assignment must be deterministic (e.g., lower-left inclusive) to avoid jittering [G].

## Subtopics

### [Re-projected Voxel Visualization](6550-re-projected-voxel-visualization.md)
An interaction technique that maps a spatial voxel to the TF domain to help users interpret local volume characteristics. It represents data value and derivatives at a query position and the eight corner samples as points in the TF domain, connecting them with edges to reflect spatial connectivity. This allows users to distinguish between homogeneous regions and boundaries based on the geometric configuration of the projected points.

### [Gaussian Widget for Transfer Function Design](6568-gaussian-widget-for-transfer-function-design.md)
An interactive interface component that allows users to define and edit color and opacity transfer functions by placing Gaussian curves over a data histogram. It differs from standard piecewise linear editors by using Gaussian distributions as basis functions, allowing for smoother blending of material properties.

## Operating characteristics
*   **Time Complexity:** The rendering of the widget is $O(1)$ with respect to the volume size, as it only processes the single voxel containing the query point and its 8 corners [S1].
*   **Memory Complexity:** $O(1)$ additional memory for the widget state and the 9 points (1 query + 8 corners) [S1].
*   **Strengths:** Provides immediate visual feedback on how local volume structure (gradient, curvature) maps to TF variables. Helps users understand the relationship between spatial features and TF parameters [S1].
*   **Limitations:** Only shows a 2D slice of the 3D TF domain; the third dimension (second derivative) is encoded via color, which may be less intuitive than a 3D plot [S1].
*   **When to use:** When users need to understand the local characteristics of the volume data (e.g., boundaries, homogeneous regions) and how they are represented in the transfer function [S1].

## Background
The 2D Transfer Function Widget was introduced in the context of interactive volume rendering using multi-dimensional transfer functions. It is part of a suite of widgets designed to facilitate dual-domain interaction, where users can query the volume and see the results in the TF domain simultaneously [S1]. The method was developed to address the difficulty of manually tuning transfer functions by providing a visual link between spatial data properties and TF parameters [S1].

## Sources
[S1] PDF Interactive Volume Rendering Using Multi-Dimensional Transfer Functions ..., https://publications.sci.utah.edu/publications/jmk01/simian-paper.pdf, fetched 2026-10-27
