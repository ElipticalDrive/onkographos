# Colormap Gaussian Addition

## Overview
Colormap Gaussian Addition is a method for generating a single Gaussian layer in a 1D transfer function where the color is determined by sampling a named colormap at a specific input value [S2]. It solves the difficulty of manually selecting pleasing color-height combinations for Gaussian peaks in volume rendering transfer functions [S2]. It differs from standard Gaussian addition by automatically mapping the peak's color to a position within a specified colormap, rather than accepting an explicit RGB tuple [S2].

## How it works
**Inputs and Outputs**
- **Inputs**:
  - `v` (float): The center value ($x_0$) of the Gaussian peak [S2].
  - `w` (float): The relative width parameter for the Gaussian decay [S2].
  - `alpha` (float, optional): The maximum alpha (opacity) height of the Gaussian [S2].
  - `colormap` (string, optional): The name of the colormap to sample from [S2].
  - `col_bounds` (array_like, optional): A pair `[min, max]` defining the range over which the colormap is spanned; if not provided, the transfer function's global bounds are used [S2].
- **Outputs**: Modifies the transfer function by adding a Gaussian layer [S2].

**Data Structures**
The method operates on a `ColorTransferFunction` object [S1, S2]. The internal representation of the transfer function is not explicitly detailed in the sources [G].

**Formula**
The Gaussian function is defined as [S2]:
$$f(x) = h \exp\left(-\frac{(x-x_0)^2}{w}\right)$$
Where:
- $x_0$ is the value `v` [S2].
- $w$ is the relative width parameter [S2].
- $h$ is the height, which is determined by the `alpha` value if provided, or derived from the colormap sampling logic if not [S2]. The color associated with this Gaussian is sampled from the colormap [S2].

**Steps**
1. Determine the normalization range for the colormap. If `col_bounds` is provided, use `[col_bounds[0], col_bounds[1]]`. Otherwise, use the transfer function's defined bounds [S2].
2. Normalize the input value `v` to the range $[0, 1]$ based on the determined bounds [G].
3. Sample the specified `colormap` at the normalized value to obtain an RGB color vector [G].
4. Construct a Gaussian layer with center `v`, width `w`, height `alpha` (or default), and the sampled RGB color [S2].
5. Add this layer to the transfer function [S2].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_colormap(tf, v, w, alpha=None, colormap='gist_stern', col_bounds=None):
    """
    Add a Gaussian layer to the transfer function with color from a colormap.
    
    Args:
        tf: ColorTransferFunction instance.
        v (float): Center value of the Gaussian.
        w (float): Relative width of the Gaussian.
        alpha (float, optional): Alpha height. Defaults to None.
        colormap (str): Name of the colormap.
        col_bounds (array_like, optional): [min, max] for colormap span.
    """
    # Determine bounds for colormap normalization
    if col_bounds is not None:
        min_val, max_val = col_bounds
    else:
        # [G] Retrieve transfer function bounds
        min_val, max_val = tf.get_bounds()
    
    # Normalize v to [0, 1]
    if max_val == min_val:
        norm_v = 0.5
    else:
        norm_v = (v - min_val) / (max_val - min_val)
    
    # Clamp normalized value to [0, 1] to handle out-of-bounds inputs
    norm_v = max(0.0, min(1.0, norm_v))
    
    # [G] Sample colormap at norm_v using standard visualization library
    color = (0.0, 0.0, 0.0)  # Standard colormap lookup returns RGB tuple
    
    # Add the Gaussian layer to the transfer function
    tf.add_gaussian(v, w, alpha, color)
```

**Parameters with Typical Values**
- `v`: Depends on the data range. In the example, `v = -7.0` for a range of `(-10.0, -5.0)` [S2].
- `w`: `0.01` is used in examples [S1, S2].
- `colormap`: `"cmyt.arbre"` or `"gist_stern"` are used in examples [S1, S2].

**Edge Cases and Pitfalls**
- **Log Space**: When rendering data in log space, the transfer function bounds and the value `v` must be specified in log space [S1].
- **Out of Bounds**: If `v` is outside the `col_bounds` or transfer function bounds, the normalized value should be clamped to $[0, 1]$ to prevent invalid colormap sampling [G].
- **Colormap Selection**: The colormap name must be valid for the underlying visualization library (e.g., `yt.visualization.color_maps` or Matplotlib) [S2].

## Operating characteristics
- **Time Complexity**: Constant time $O(1)$ for adding a single layer [G].
- **Memory Complexity**: Constant space $O(1)$ for the layer data [G].
- **Strengths**: Simplifies the creation of visually distinct Gaussian peaks by leveraging existing colormaps [G].
- **Limitations**: Only adds a single Gaussian layer per call [S2]. The color is fixed by the colormap position, offering less direct control than specifying RGB manually [G].
- **When to use**: When a quick, visually pleasing Gaussian peak is needed with a color that fits a broader colormap scheme [G].
- **When not to use**: For multiple Gaussians, `add_layers` is available [S2].

## Background
This method is part of the `yt` visualization toolkit, specifically within the `ColorTransferFunction` class [S1, S2]. It was introduced to address the challenges of constructing pleasing Gaussians in transfer functions by automating color selection from standard colormaps [S2]. It is demonstrated in volume rendering workflows for scientific data, such as astrophysical simulations [S1].

## Sources
[S1] 3D Visualization and Volume Rendering, https://yt-project.org/doc/visualizing/volume_rendering.html, fetched 2026-10-28
[S2] ColorTransferFunction yt.visualization.volume_rendering.transfer_functions.ColorTransferFunction, https://yt-project.org/doc/reference/api/yt.visualization.volume_rendering.transfer_functions.html, fetched 2026-10-29
[S3] https://yt-project.org/docs/dev/visualizing/volume_rendering.html#volume-rendering-components, https://yt-project.org/docs/dev/visualizing/volume_rendering.html, fetched 2026-10-29
