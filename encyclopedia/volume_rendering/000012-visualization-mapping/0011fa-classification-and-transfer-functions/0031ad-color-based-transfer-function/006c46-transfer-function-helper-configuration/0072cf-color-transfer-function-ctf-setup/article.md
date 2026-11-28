# Color Transfer Function (CTF) Setup

Also called: applyCanvasCTFToActor

## Overview
Configures the color mapping for a volume rendering actor by defining RGB values for specific scalar ranges [S1, unofficial]. It differs from generic transfer function construction by explicitly mapping normalized color values to the scalar data range using linear interpolation, often within a helper configuration context [S1, unofficial].

## How it works
**Inputs:**
- `actor`: The volume rendering actor.
- `minPixel`: Minimum scalar value of the volume data [S1, unofficial].
- `maxPixel`: Maximum scalar value of the volume data [S1, unofficial].
- `controlPoints`: Array of objects, each containing:
  - `color`: Hex string representing the desired color.
  - `colorValue`: Normalized value in the range [0, 1] [S1, unofficial].

**Outputs:**
- Updated `actor` with a configured `vtkColorTransferFunction`.

**Data Structures:**
- `vtkColorTransferFunction`: A VTK object storing RGB points.
- `controlPoint`: Object with `color` (string) and `colorValue` (float).

**Steps:**
1. Retrieve the volume property from the actor.
2. Get or instantiate a `vtkColorTransferFunction` (CTF).
3. Clear any existing points in the CTF.
4. For each `controlPoint` in `controlPoints`:
   a. Convert the hex `color` to normalized RGB components $(r, g, b)$ where each component is in $[0, 1]$.
   b. Compute the scalar value $s$ using linear interpolation:
      $$s = \text{minPixel} + \text{colorValue} \times (\text{maxPixel} - \text{minPixel})$$
   c. Add the point to the CTF using `addRGBPointLong(s, r, g, b, 0.5, 0)`. The last two arguments are opacity and opacity gradient, fixed at 0.5 and 0 respectively [S1, unofficial].
5. Assign the CTF to the volume property using `property.setRGBTransferFunction(0, ctf)`.
6. Call `property.modified()` to notify the renderer.
7. If the actor has a mapper with input data, call `inputData.modified()` to force a render update [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def setup_ctf(actor, min_pixel, max_pixel, control_points):
    # Retrieve volume property
    property = actor.getProperty()
    
    # Get or create CTF
    ctf = property.getRGBTransferFunction(0)
    if ctf is None:
        ctf = vtkColorTransferFunction()
        
    # Clear existing points
    ctf.removeAllPoints()
    
    # Process control points
    for cp in control_points:
        # Convert hex to normalized RGB
        hex_color = cp['color']
        r = int(hex_color[1:3], 16) / 255.0
        g = int(hex_color[3:5], 16) / 255.0
        b = int(hex_color[5:7], 16) / 255.0
        
        # Compute scalar
        color_value = cp['colorValue']
        scalar = min_pixel + color_value * (max_pixel - min_pixel)
        
        # Add point with fixed opacity parameters
        ctf.addRGBPointLong(scalar, r, g, b, 0.5, 0)
        
    # Assign CTF to property
    property.setRGBTransferFunction(0, ctf)
    property.modified()
    
    # Force update if mapper exists
    if actor.getMapper() and actor.getMapper().getInputData():
        actor.getMapper().getInputData().modified()
```

**Parameters:**
- `minPixel`, `maxPixel`: Define the scalar range of the volume data [S1, unofficial].
- `controlPoints`: Array of color mappings [S1, unofficial].

**Edge Cases:**
- Empty `controlPoints` array results in a blank transfer function.
- Hex color strings must be valid 6-character strings.
- `colorValue` outside [0, 1] leads to scalars outside the data range, potentially causing undefined behavior or clipping.

## Subtopics

### Default Gaussian Transfer Function Construction
Generates a default volume rendering transfer function using Gaussian layers sampled from a colormap, with alpha scaling for contrast [S2, unofficial]. It creates a `ColorTransferFunction` with 10 Gaussian layers, sampling colors from the 'nipy_spectral' colormap [S2, unofficial]. It scales the alpha channel (y) of the last Gaussian layer to enhance contrast using the formula:
$$y_{new} = y_{old} \times 2 \times factor$$
where $factor = \frac{N}{\sum y}$, $N$ is the number of samples in the last layer's y-array, and $\sum y$ is the sum of its y-values [S2, unofficial]. This method is used when a default, visually balanced transfer function is needed without manual configuration [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of control points $O(N)$ for setup [S1, unofficial].
- **Memory Complexity:** Depends on the number of control points and the internal representation of the CTF [S1, unofficial].
- **Strengths:** Simple, explicit control over color mapping; integrates well with VTK-based renderers [S1, unofficial].
- **Limitations:** Requires manual definition of control points; does not automatically adapt to data statistics [S1, unofficial].
- **When to use:** When specific colors need to be mapped to specific scalar ranges in a VTK-based volume renderer [S1, unofficial].
- **When not to use:** When automatic transfer function generation based on data histograms is preferred [S2, unofficial].

## Background
Originated in the context of DICOM viewer applications using VTK.js and Cornerstone3D for volume rendering [S1, unofficial]. It is part of the visualization mapping pipeline, specifically within classification and transfer function configuration [S1, unofficial]. The method is implemented in projects like `dicom-viewer` to allow users to adjust display parameters for medical imaging [S1, unofficial]. The Gaussian variant is associated with the `yt` project for scientific visualization [S2, unofficial].

## Sources
[S1] yuxi-ovo/dicom-viewer, https://github.com/yuxi-ovo/dicom-viewer, fetched 2026-10-27 (unofficial)
[S2] Volume rendering colormap is not directly exposed, https://github.com/yt-project/yt/issues/2804, fetched 2026-10-27 (unofficial)
