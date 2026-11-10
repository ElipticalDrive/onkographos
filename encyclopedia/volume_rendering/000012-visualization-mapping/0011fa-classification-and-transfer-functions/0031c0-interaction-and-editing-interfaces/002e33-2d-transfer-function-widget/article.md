# 2D Transfer Function Widget

Also called: 2D TF Display, TF Widget, Transfer Function Widget

## Overview
The 2D Transfer Function Widget is an interactive visualization component that maps a spatial query in a volume to a 2D slice of the multi-dimensional transfer function (TF) domain, allowing users to inspect local data characteristics and adjust classification parameters [S1]. It solves the ambiguity of how specific voxel structures map to TF variables by displaying the queried point and its eight voxel corners as points in the TF space, connected by edges reflecting spatial connectivity [S1]. Unlike standard point probes, it visualizes the local gradient and curvature structure, enabling users to distinguish between homogeneous regions and boundaries based on the geometric configuration of the projected points [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A query point $\mathbf{p}$ in the volume domain, obtained via a Data Probe or Clipping Plane widget [S1].
*   **Output:** A 2D graphical representation in the TF domain showing the query point, eight voxel corner points, connecting edges, and a texture-mapped slice of the 3D Lookup Table (LUT) [S1].

**Data Structures:**
The widget is implemented as a basic frame widget with the following layout [S1]:
*   **Horizontal Axis:** Data value (scalar intensity).
*   **Vertical Axis:** Gradient magnitude $\|\nabla f\|$.
*   **Inner Plane:** A polygon texture-mapped with a slice through the 3D LUT containing the full 3D transfer function [S1].
*   **Controls:**
    *   Balls at the four corners of the widget frame for resizing [S1].
    *   Bars on the edges for translating the widget's position [S1].
*   **Visual Elements:**
    *   **Query Ball:** Represents the data value and derivatives at the query point $\mathbf{p}$ [S1].
    *   **Corner Balls:** Represent the eight data sample points at the corners of the voxel containing $\mathbf{p}$ [S1].
    *   **Edges:** Connect the corner balls to reflect the spatial connectivity of the voxel corners [S1].
    *   **Color Mapping:** The balls are colormapped based on the second derivative value at that location: negative values are blue, zero is white, and positive values are yellow [S1].

**Algorithm:**
1.  **Query Acquisition:** Obtain the query point $\mathbf{p}$ from the volume via a Data Probe or Clipping Plane interaction [S1].
2.  **Voxel Identification:** Identify the voxel $V$ containing $\mathbf{p}$ [S1].
3.  **Feature Extraction:** Calculate the data value $v$, gradient magnitude $g = \|\nabla f\|$, and second derivative $s$ for the query point $\mathbf{p}$ [S1].
4.  **Corner Extraction:** For each of the eight corners $c_i$ of voxel $V$ (where $i \in \{0, \dots, 7\}$), calculate the data value $v_i$, gradient magnitude $g_i = \|\nabla f(c_i)\|$, and second derivative $s_i$ [S1].
5.  **TF Mapping:** Map the query point to coordinates $(v, g)$ in the widget [S1]. Map each corner $c_i$ to coordinates $(v_i, g_i)$ in the widget [S1].
6.  **Rendering:**
    *   Render a ball at $(v, g)$ for the query point [S1].
    *   Render balls at $(v_i, g_i)$ for each corner [S1].
    *   Color each ball (query and corners) based on its second derivative value $s$ or $s_i$:
        *   If $s < 0$, color is blue.
        *   If $s = 0$, color is white.
        *   If $s > 0$, color is yellow.
    *   Draw edges connecting the corner balls to reflect the spatial connectivity of the voxel [S1].
    *   Display the inner plane as a texture-mapped slice of the 3D LUT [S1].
7.  **Interaction:** Allow the user to observe how TF variables change near the probe and adjust the TF slice or parameters via the widget controls [S1].

**Parameters:**
*   **Second Derivative Color Map:** Fixed mapping of sign to color (Blue/White/Yellow) [S1].
*   **TF Axes:** Horizontal = Data Value, Vertical = Gradient Magnitude [S1].

**Edge Cases and Pitfalls:**
*   The second derivative axis is not explicitly represented in the 2D widget; it is encoded via ball color [S1].
*   The widget displays only a single slice of the 3D LUT; other dimensions are controlled by other sub-widgets [S1].

## Subtopics

### [Re-projected Voxel Visualization](6550-re-projected-voxel-visualization.md)
An interaction technique that maps a spatial voxel to the TF domain to help users interpret local volume characteristics. It differs from the standard widget by explicitly visualizing the local gradient and curvature structure through the geometric configuration of projected points and edges, allowing distinction between homogeneous regions and boundaries.

### [Gaussian Widget for Transfer Function Design](6568-gaussian-widget-for-transfer-function-design.md)
An interactive interface component that allows users to define and edit color and opacity transfer functions by placing Gaussian curves over a data histogram. It differs from the 2D TF Widget by focusing on histogram-based Gaussian curve editing rather than spatial re-projection of voxel corners.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Provides dual-domain interaction, helping users understand how local volume structures map to TF variables [S1]. Allows manual setting and alteration of the transfer function based on visual feedback from queried voxels [S1].
*   **Limitations:** Only displays a single slice of the 3D LUT; other dimensions require other sub-widgets [S1].
*   **When to use:** When users need to inspect the relationship between spatial voxel structures (homogeneity vs. boundaries) and transfer function variables (data value, gradient) [S1].
*   **When not to use:** When the full 3D transfer function space needs to be visualized simultaneously without slicing [S1].

## Background
The 2D Transfer Function Widget was introduced in the context of interactive volume rendering using multi-dimensional transfer functions [S1]. It is part of a system that includes other widgets such as the Data Probe and Clipping Plane, designed to facilitate dual-domain interaction for volume rendering tasks, such as rendering CT tooth datasets [S1].

## Sources
[S1] Interactive Volume Rendering Using Multi-Dimensional Transfer Functions, https://publications.sci.utah.edu/publications/jmk01/simian-paper.pdf, fetched 2026-10-27
