# Colormap Gaussian Addition


## Overview
Colormap Gaussian Addition is a method for generating a single Gaussian layer in a 1D transfer function where the color is determined by sampling a named colormap at a specific input value. It solves the difficulty of manually selecting pleasing color-height combinations for Gaussian peaks in volume rendering transfer functions. It differs from standard Gaussian addition by automatically mapping the peak's color to a position within a specified colormap, rather than accepting an explicit RGB tuple.

## How it works
**Inputs and Outputs**
- **Inputs**:
  - `v` (float): The center value (x-coordinate) of the Gaussian peak.
  - `w` (float): The relative width parameter for the Gaussian decay.
  - `alpha` (float, optional): The maximum alpha (opacity) height of the Gaussian. If not provided, defaults are used.
  - `colormap` (string, optional): The name of the colormap to sample from.
  - `col_bounds` (array_like, optional): A pair `[min, max]` defining the range over which the colormap is spanned. If not provided, the transfer function's global bounds are used.
- **Outputs**: Modifies the transfer function by adding a Gaussian layer.

**Data Structures**
The method operates on a `ColorTransferFunction` object. The internal representation of the transfer function is not explicitly detailed in the sources, but it stores layers of Gaussian functions defined by center, width, height, and color.

**Formula**
The Gaussian function is defined as [S1]:
$$f(x) = h \exp\left(-\frac{(x-x_0)^2}{w}\right)$$
Where:
- $x_0$ is the value `v`.
- $w$ is the relative width parameter.
- $h$ is the height, which is determined by the alpha value `alpha` if provided, or derived from the colormap sampling logic if not. The color associated with this Gaussian is sampled from the colormap.

**Steps**
1. Determine the normalization range for the colormap. If `col_bounds` is provided, use `[col_bounds[0], col_bounds[1]]`. Otherwise, use the transfer function's defined bounds.
2. Normalize the input value `v` to the range $[0, 1]$ based on the determined bounds.
3. Sample the specified `colormap` at the normalized value to obtain an RGB color vector.
4. Construct a Gaussian layer with center `v`, width `w`, height `alpha` (or default), and the sampled RGB color.
5. Add this layer to the transfer function.

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
        # Get bounds from the transfer function object
        min_val, max_val = tf.get_bounds()
    
    # Normalize v to [0, 1]
    # Handle edge case where min_val == max_val to avoid division by zero
    if max_val == min_val:
        norm_v = 0.5
    else:
        norm_v = (v - min_val) / (max_val - min_val)
    
    # Clamp normalized value to [0, 1] to handle out-of-bounds inputs
    norm_v = max(0.0, min(1.0, norm_v))
    
    # Sample the colormap to get RGB color
    # Note: Implementation depends on the colormap library (e.g., matplotlib)
    # Assuming a function get_colormap_color(name, normalized_value) -> (r, g, b)
    color = get_colormap_color(colormap, norm_v)
    
    # Add the Gaussian layer to the transfer function
    # The height 'alpha' is used if provided; otherwise, a default height is applied
    tf.add_gaussian(v, w, alpha, color)
```

**Parameters with Typical Values**
- `v`: Depends on the data range. In the example, `v = -7.0` for a range of `(-10.0, -5.0)` [S1].
- `w`: `0.01` is used in examples [S1, S2].
- `colormap`: `"cmyt.arbre"` or `"gist_stern"` are used in examples [S1, S2].

**Edge Cases and Pitfalls**
- **Log Space**: When rendering data in log space, the transfer function bounds and the value `v` must be specified in log space [S2].
- **Out of Bounds**: If `v` is outside the `col_bounds` or transfer function bounds, the normalized value should be clamped to $[0, 1]$ to prevent invalid colormap sampling.
- **Colormap Selection**: The colormap name must be valid for the underlying visualization library (e.g., `yt.visualization.color_maps` or Matplotlib) [S1].

## Operating characteristics
- **Time Complexity**: Constant time $O(1)$ for adding a single layer.
- **Memory Complexity**: Constant space $O(1)$ for the layer data.
- **Strengths**: Simplifies the creation of visually distinct Gaussian peaks by leveraging existing colormaps.
- **Limitations**: Only adds a single Gaussian layer per call. The color is fixed by the colormap position, offering less direct control than specifying RGB manually.
- **When to use**: When a quick, visually pleasing Gaussian peak is needed with a color that fits a broader colormap scheme.
- **When not to use**: When precise color control is required or when multiple Gaussians need to be added simultaneously (use `add_layers` instead) [S1].

## Background
This method is part of the `yt` visualization toolkit, specifically within the `ColorTransferFunction` class. It was introduced to address the challenges of constructing pleasing Gaussians in transfer functions by automating color selection from standard colormaps [S1]. It is commonly used in volume rendering workflows for scientific data, such as astrophysical simulations [S2].

## Sources
[S1] ColorTransferFunction yt.visualization.volume_rendering.transfer_functions.ColorTransferFunction, https://yt-project.org/doc/reference/api/yt.visualization.volume_rendering.transfer_functions.html, fetched 2026-10-29
[S2] https://yt-project.org/docs/dev/visualizing/volume_rendering.html#volume-rendering-components, https://yt-project.org/docs/dev/visualizing/volume_rendering.html, fetched 2026-10-29
