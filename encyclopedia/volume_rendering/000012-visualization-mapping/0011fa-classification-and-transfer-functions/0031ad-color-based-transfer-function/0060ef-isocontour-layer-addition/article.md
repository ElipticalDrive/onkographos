# Isocontour Layer Addition

Also called: CTF, ColorTransferFunction, Evenly Spaced Isocontour Layering

## Overview
Isocontour Layer Addition is a method for constructing a `ColorTransferFunction` by placing evenly spaced discrete isocontours along the field value range and sampling a colormap to assign colors to each layer [S1, S2]. It solves the problem of visualizing nested structures in simulation data by creating distinct, discrete visual layers [S1]. It differs from continuous transfer function construction by explicitly defining discrete isocontours rather than smooth gradients, and from single-layer sampling by distributing multiple layers across the range [S1, S2].

## How it works
**Inputs and Outputs:**
- **Inputs:**
  - `bounds`: A tuple `(min, max)` defining the field value range [S2].
  - `n`: An integer specifying the number of evenly spaced isocontours to add [S1, S2].
  - `colormap`: A string identifier for the colormap to sample (e.g., "cmyt.arbre") [S1, S2].
  - `log_space`: A boolean or context indicating if the data is log-scaled, requiring the transfer function to be specified in log space [S1].
- **Output:** A configured `ColorTransferFunction` object with `n` discrete layers added [S1, S2].

**Data Structures:**
The `ColorTransferFunction` maintains an internal representation of layers. Each layer corresponds to an isocontour. The exact internal layout of the `ColorTransferFunction` object is not specified in the sources [G: Implementation requires a container for layer definitions, each storing position, color, and width parameters].

**Steps:**
1. **Initialize Transfer Function:** Create a `ColorTransferFunction` instance. If the data is log-scaled, the `bounds` must be converted to log space (e.g., using $\log_{10}$) before initialization [S1].
   $$ \text{log\_bounds} = (\log_{10}(\text{min}), \log_{10}(\text{max})) $$
2. **Add Layers:** Call the `add_layers` method with the count `n` and the `colormap` string. This method adds `n` evenly spaced isocontours along the transfer function range, sampling the specified colormap to determine the color of each layer [S1, S2].
3. **Assign to Source:** Assign the created `ColorTransferFunction` to the source's transfer function property (e.g., `source.tfh.tf`) [S1].
4. **Set Bounds:** Ensure the source's transfer function helper bounds are set to the original (non-log) bounds if required by the rendering pipeline [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def isocontour_layer_addition(bounds, n, colormap, log_scale=True):
    """
    Constructs a ColorTransferFunction with evenly spaced isocontours.
    
    Args:
        bounds (tuple): (min, max) field values.
        n (int): Number of evenly spaced isocontours.
        colormap (str): Name of the colormap to sample.
        log_scale (bool): If True, convert bounds to log10 space for TF initialization.
        
    Returns:
        ColorTransferFunction: Configured transfer function.
    """
    # Step 1: Initialize ColorTransferFunction
    # If log_scale is True, bounds must be in log space for the TF
    if log_scale:
        log_bounds = (np.log10(bounds[0]), np.log10(bounds[1]))
    else:
        log_bounds = bounds
        
    # Create the transfer function object
    # Note: The exact class name and constructor may vary by library (e.g., yt.ColorTransferFunction)
    tf = ColorTransferFunction(log_bounds)
    
    # Step 2: Add evenly spaced isocontours
    # This samples the colormap to assign colors to each layer
    tf.add_layers(n, colormap=colormap)
    
    return tf

# Example Usage:
# bounds = (3e-31, 5e-27)
# tf = isocontour_layer_addition(bounds, n=5, colormap="cmyt.arbre", log_scale=True)
# source.tfh.tf = tf
# source.tfh.bounds = bounds
```

**Parameters:**
- `n`: Integer count of evenly spaced isocontours. Typical value: 5 [S1, S2].
- `colormap`: String identifier. Example: "cmyt.arbre" [S1, S2].
- `bounds`: Tuple of (min, max) field values. Example: `(3e-31, 5e-27)` [S1, S2].

**Edge Cases and Pitfalls:**
- **Log Space:** If the data is log-scaled, the transfer function bounds must be specified in log space. Failure to do so results in incorrect layer placement [S1].
- **Bounds Mismatch:** The source's transfer function helper bounds should be set to the original (non-log) bounds if the rendering pipeline expects them in linear space [S1].

## Subtopics

### [Windowing](6611-windowing.md)
Adjusts the visible range of field values. Use when focusing on a specific subset of data.

### [Colormap Application for Visualization](7714-colormap-application-for-visualization.md)
Applies a colormap to field values. Use for general color mapping.

### [Default Spine-Focused Transfer Function](11520-default-spine-focused-transfer-function.md)
Uses a default spine-focused approach. Use for spine-like structures.

### [Color Affine Transform](12832-color-affine-transform.md)
Applies affine transforms to colors. Use for color adjustments.

### [Iterative Color Matching](12841-iterative-color-matching.md)
Iteratively matches colors. Use for precise color matching.

### [Palette Remapping](12863-palette-remapping.md)
Remaps color palettes. Use for palette changes.

### [Color Composition](13205-color-composition.md)
Composes colors. Use for multi-layer color blending.

### [Reference Image Guided Colorization](13231-reference-image-guided-colorization.md)
Uses reference images for colorization. Use for style transfer.

### [HSV Transfer Function Mapping](13588-hsv-transfer-function-mapping.md)
Maps transfer functions in HSV space. Use for hue control.

### [Combining Lightening and Darkening](14287-combining-lightening-and-darkening.md)
Combines lightening and darkening. Use for contrast enhancement.

### [Gradient-Based Color Table Filtering](16456-gradient-based-color-table-filtering.md)
Filters color tables based on gradients. Use for gradient-based selection.

### [Dynamic Transfer Function Adjustment](18245-dynamic-transfer-function-adjustment.md)
Adjusts transfer functions dynamically. Use for interactive visualization.

### [Transfer Function Helper Configuration](21063-transfer-function-helper-configuration.md)
Configures transfer function helpers. Use for helper setup.

### [Interior-Emphasized Transfer Function Style](21201-interior-emphasized-transfer-function-style.md)
Emphasizes interior features. Use for interior visualization.

### [Color Mapping Parameters Configuration](22483-color-mapping-parameters-configuration.md)
Configures color mapping parameters. Use for parameter tuning.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Easy to create discrete isocontours; works well for visualizing nested structures [S1].
- **Limitations:** Limited to evenly spaced layers; requires manual selection of colormap and layer count [S1, S2].
- **When to Use:** When visualizing nested structures with discrete layers is desired [S1].
- **When Not to Use:** When continuous features or non-evenly spaced layers are required [S1].

## Background
The method is part of the `yt` library's volume rendering capabilities, specifically within the `ColorTransferFunction` interface [S1, S2]. It was introduced as a standard way to map dataset field values to colors, brightnesses, and opacities in rendered rays [S1]. The `add_layers` function is described as the easiest way to create a `ColorTransferFunction` for volume rendering [S1, S2].

## Sources
[S1] 3D Visualization and Volume Rendering, https://yt-project.org/doc/visualizing/volume_rendering.html, fetched 2026-10-28
[S2] https://yt-project.org/docs/dev/visualizing/volume_rendering.html#volume-rendering-components, https://yt-project.org/docs/dev/visualizing/volume_rendering.html, fetched 2026-10-29
