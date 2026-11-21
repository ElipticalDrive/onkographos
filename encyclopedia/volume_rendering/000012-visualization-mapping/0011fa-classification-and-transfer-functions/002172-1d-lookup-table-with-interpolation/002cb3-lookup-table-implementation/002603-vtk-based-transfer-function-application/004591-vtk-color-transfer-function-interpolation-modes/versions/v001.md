# VTK Color Transfer Function Interpolation Modes

Also called: Color Space Interpolation, Color Transfer Function Interpolation Modes, Diverging interpolation, HSVWrap, Step interpolation, Transfer Function Color Mapping

## Overview
A method for mapping scalar values to colors using various interpolation spaces (RGB, HSV, CIELAB, PROLAB, Diverging, Step) within a color transfer function. It sets apart from standard linear RGB interpolation by offering perceptually uniform spaces (CIELAB) and specialized behaviors (Diverging, Step) to enhance visual clarity in volume rendering.

## How it works
**Inputs and Outputs:**
- Input: A scalar value $s$ to be mapped, and a set of control points $(x_i, c_i)$ where $x_i$ is the scalar position and $c_i$ is the color $(R_i, G_i, B_i)$ or $(H_i, S_i, V_i)$ defined at that position.
- Output: The interpolated color $(R, G, B)$ and opacity $\alpha$ at $s$.

**Data Structures:**
The `vtkColorTransferFunction` maintains a list of nodes. Each node contains:
- `position`: float, the scalar value.
- `color`: array of 3 floats, the RGB or HSV values.
- `opacity`: float, the opacity at this node (handled by `vtkPiecewiseFunction`, but relevant for the full transfer function context).

**Interpolation Modes:**
The mode is set via `SetColorSpace()`. The available modes are:
1.  **RGB**: Standard linear interpolation in RGB space.
2.  **HSV**: Interpolation in Hue, Saturation, Value space.
3.  **CIELAB**: Interpolation in CIELAB space.
4.  **PROLAB**: Interpolation in PROLAB space.
5.  **Diverging**: A special mode where colors pass through white when interpolating between two saturated colors.
6.  **Step**: The color of an interval is the color of the second (upper) color of that interval.

**HSVWrap Behavior:**
In HSV mode, the `HSVWrap` flag controls hue interpolation:
-   **HSVWrapOn**: Takes the shortest path in the hue circle, potentially crossing the 0/360 boundary.
-   **HSVWrapOff**: Does not cross the 0/360 boundary, matching the functionality of `vtkLookupTable`.

**CIELAB/CIEDE2000 Behavior:**
In Lab/CIEDE2000 mode, interpolation takes the shortest path in the Lab color space with respect to the CIE Delta E 2000 color distance measure.

**Diverging Behavior:**
When interpolating between two saturated colors, the resulting color passes through white.

**Step Behavior:**
The color of an interval is set to the color of the second (upper) color of that interval.

**Algorithm:**
1.  Determine the mode $M$ set by `SetColorSpace()`.
2.  Find the interval $[x_i, x_{i+1}]$ containing the input scalar $s$.
3.  If $s < x_0$, return the color of the first node (or black if clamping is off, though the source notes mention "upper range value... mapped to last node color. If off, values outside the range are mapped to black" for the lookup table context, implying clamping behavior).
4.  If $s > x_{n-1}$, return the color of the last node.
5.  Calculate the interpolation parameter $t = \frac{s - x_i}{x_{i+1} - x_i}$.
6.  Interpolate the color components based on mode $M$:
    -   **RGB**: $C = (1-t)C_i + tC_{i+1}$ for each component $C \in \{R, G, B\}$.
    -   **HSV**:
        -   Convert $C_i$ and $C_{i+1}$ to HSV.
        -   If `HSVWrapOn`, compute the shortest hue difference $\Delta H$. If $|\Delta H| > 180$, adjust by $\pm 360$ to find the shortest path.
        -   If `HSVWrapOff`, do not adjust for the 0/360 boundary.
        -   Interpolate H, S, V linearly: $H = (1-t)H_i + tH_{i+1}$, etc.
        -   Convert back to RGB.
    -   **CIELAB/PROLAB**:
        -   Convert $C_i$ and $C_{i+1}$ to the respective space.
        -   Interpolate linearly in that space.
        -   Convert back to RGB.
    -   **Diverging**:
        -   Detect if the two colors are "saturated".
        -   If so, interpolate through white. This likely involves a two-step interpolation or a specific formula that ensures the midpoint is white. The source states "colors will pass through white".
    -   **Step**:
        -   Return $C_{i+1}$.

**Parameters:**
-   `ColorSpace`: Enum or integer constant for RGB, HSV, CIELAB, PROLAB, Diverging, Step.
-   `HSVWrap`: Boolean, default likely Off.

**Edge Cases:**
-   **Hue Crossing**: In HSV mode, if `HSVWrap` is Off, interpolation may take a long path around the hue circle if the hues are near 0/360.
-   **Saturated Colors in Diverging**: The definition of "saturated" is not explicitly detailed in the source, but the behavior is to pass through white.
-   **Out of Range**: Values outside the defined range are mapped to the first or last node color, or black if clamping is off.

## Subtopics

### [Piecewise Hermite Color Transfer Function](11553-piecewise-hermite-color-transfer-function.md)
A color mapping technique that uses modified piecewise Hermite functions to interpolate RGB or HSV values between user-defined nodes, supporting interpolation ranging from piecewise constant to piecewise linear.

## Operating characteristics
-   **Time Complexity**: Linear in the number of nodes for finding the interval, then constant time for interpolation.
-   **Memory Complexity**: Linear in the number of nodes.
-   **Strengths**: Provides perceptually uniform interpolation (CIELAB), handles hue wrapping correctly (HSV), and offers specialized modes for specific visualization needs (Diverging, Step).
-   **Limitations**: HSV interpolation can be unintuitive if `HSVWrap` is Off. Diverging mode requires detection of saturation.

## Background
Originated in the Visualization Toolkit (VTK) library. The `vtkColorTransferFunction` class provides these interpolation modes to allow users to create more effective and perceptually accurate color maps for volume rendering and other visualization tasks.

## Sources
[S1] vtkColorTransferFunctions, http://www.vtk.org/doc/nightly/html/classvtkColorTransferFunction.html, fetched 2026-10-29
