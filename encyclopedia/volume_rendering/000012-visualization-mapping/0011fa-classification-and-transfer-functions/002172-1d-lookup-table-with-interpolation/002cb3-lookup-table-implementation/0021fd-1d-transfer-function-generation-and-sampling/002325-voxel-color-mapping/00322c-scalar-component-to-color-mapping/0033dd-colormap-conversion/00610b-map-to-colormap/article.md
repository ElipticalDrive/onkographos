# Map to Colormap

## Overview
`map_to_colormap` maps a continuous colormap directly to a specified range of scalar values (typically density), assigning a single constant alpha value to all values within that range. Unlike layered transfer function methods that distribute opacity across different scalar bands, this method allows all density values in the dataset to contribute to the volume rendering simultaneously at a uniform visibility level [S1, S2]. It is primarily used for projection-like visualizations where the goal is to display the full dynamic range of a field without layering effects [S1, S2].

## How it works
**Inputs:**
- `min_val`: The lower bound of the scalar range (e.g., in log space) [S1, S2].
- `max_val`: The upper bound of the scalar range (e.g., in log space) [S1, S2].
- `colormap`: A string identifier for the colormap to use (e.g., `"cmyt.arbre"`, `"turbo"`) [S1, S2].
- `scale_func`: A callable function that normalizes or scales input values to the range $[0, 1]$ for colormap indexing [S1, S2].

**Outputs:**
- A configured transfer function object where the specified scalar range is mapped to the colors of the given colormap with a constant alpha [S1, S2].

**Data Structures:**
The method operates on a `ColorTransferFunction` object. The internal state includes:
- `tf`: The transfer function definition, storing the mapping rules [S1, S2].
- `bounds`: The scalar range bounds associated with the transfer function [S1, S2].

**Steps:**
1. Initialize the transfer function with the log-transformed bounds of the data range: $bounds_{log} = [\log_{10}(\text{min\_val}), \log_{10}(\text{max\_val})]$ [S1, S2].
2. Define a scaling function. The standard example uses a linear ramp that normalizes values relative to their minimum and maximum within the provided range:
   $$ \text{scale}(v) = \frac{v - \min(v)}{\max(v) - \min(v)} $$
   where $v$ represents the input scalar values [S1, S2].
3. Call `map_to_colormap` with the log-transformed `min_val`, `max_val`, the `colormap` string, and the `scale_func` [S1, S2].
4. The implementation maps each scalar value $s$ in the range $[\text{min\_val}, \text{max\_val}]$ to a normalized value $n \in [0, 1]$ using `scale_func(s, min_val, max_val)`.
5. The color $C$ is retrieved from the colormap at index $n$.
6. The alpha value is set to a constant (typically 1.0 or a user-defined constant, though the sources imply a single alpha for the whole range) for all values in the range [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def linramp(vals, minval, maxval):
    """
    Scales values to [0, 1] based on the min and max of the input array.
    Note: In the source example, vals.min() and vals.max() refer to the 
    global min/max of the dataset or the specific range bounds if passed 
    appropriately. The source implementation uses:
    return (vals - vals.min()) / (vals.max() - vals.min())
    """
    # Guard against division by zero if minval == maxval
    if maxval == minval:
        return np.zeros_like(vals)
    return (vals - minval) / (maxval - minval)

def map_to_colormap(min_val, max_val, colormap, scale_func):
    """
    Maps a colormap to a scalar range.
    
    Parameters:
    min_val (float): Lower bound of the scalar range (e.g., np.log10(3e-31)).
    max_val (float): Upper bound of the scalar range (e.g., np.log10(5e-27)).
    colormap (str): Name of the colormap (e.g., "cmyt.arbre").
    scale_func (callable): Function to normalize values to [0, 1].
    
    Returns:
    None (Modifies the transfer function object in place in the source context).
    """
    # 1. Initialize transfer function with log bounds
    # The source initializes: tf = yt.ColorTransferFunction(np.log10(bounds))
    # where bounds = (min_val, max_val) if already in log space, or 
    # bounds are the linear values and np.log10 is applied.
    # Here we assume min_val and max_val are already the log values to be used as bounds.
    tf = ColorTransferFunction([min_val, max_val])
    
    # 2. Apply the colormap mapping
    # The source calls: tf.map_to_colormap(min_val, max_val, colormap, scale_func)
    # Internally, this likely:
    # a. Registers a mapping rule for the range [min_val, max_val].
    # b. Sets alpha to a constant value for this range.
    # c. Links the color output to the colormap indexed by scale_func.
    
    # Pseudocode for the internal logic of map_to_colormap:
    # For a given scalar value s:
    #   if min_val <= s <= max_val:
    #       normalized = scale_func(s, min_val, max_val)
    #       color = get_color_from_colormap(colormap, normalized)
    #       alpha = constant_alpha # e.g., 1.0
    #   else:
    #       color = (0, 0, 0, 0) # Transparent
    
    # Note: The exact internal implementation details of yt's ColorTransferFunction
    # are not fully exposed in the source text, but the API usage is clear.
    # The scale_func in the example is defined as:
    # def linramp(vals, minval, maxval):
    #     return (vals - vals.min()) / (vals.max() - vals.min())
    # This implies the scale_func might operate on the entire dataset array 
    # or the range bounds. The source example passes minval/maxval to the function.
    
    # To implement this generically:
    tf.add_colormap_mapping(
        range_min=min_val,
        range_max=max_val,
        colormap_name=colormap,
        scale_function=scale_func,
        alpha=1.0 # Constant alpha as implied by "single alpha value"
    )
    
    return tf
```

**Parameters with Typical Values:**
- `min_val`: `np.log10(3e-31)` [S1, S2].
- `max_val`: `np.log10(5e-27)` [S1, S2].
- `colormap`: `"cmyt.arbre"` [S1, S2].
- `scale_func`: `linramp` (linear ramp) [S1, S2].

**Edge Cases and Pitfalls:**
- **Log Space:** If the data is rendered in log space, the `min_val` and `max_val` must be provided in log space (e.g., `np.log10(bounds)`) [S1, S2].
- **Division by Zero:** If `min_val` equals `max_val`, the scaling function will divide by zero. A guard should be added [G].
- **Alpha Uniformity:** Since all values in the range share a single alpha, overlapping structures may not show depth cues as effectively as layered methods [S1, S2].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of voxels sampled, as each voxel's scalar value is mapped to a color via the scale function and colormap lookup [S1, S2].
- **Memory Complexity:** Constant, dependent on the size of the colormap and the transfer function object [S1, S2].
- **Strengths:** Simple to configure; allows visualization of the entire dynamic range of a dataset without manual tuning of multiple transfer function bands [S1, S2].
- **Limitations:** Lacks the ability to highlight specific density ranges with varying opacity; may result in visual clutter if the colormap does not distinguish well between similar density values [S1, S2].
- **When to Use:** When generating projection-like images where the goal is to show the distribution of all values in a field, similar to standard projection plots [S1, S2].

## Background
The method is part of the `yt` visualization framework's volume rendering capabilities [S1, S2]. It was introduced to provide an alternative to layered transfer functions, enabling users to create visualizations that resemble standard projection plots but with the flexibility of volume rendering techniques [S1, S2]. The specific example using `cmyt.arbre` colormap and log-transformed density bounds is documented in the `yt` project's 3D Visualization and Volume Rendering guide [S1, S2].

## Sources
[S1] 3D Visualization and Volume Rendering, https://yt-project.org/doc/visualizing/volume_rendering.html, fetched 2026-10-28
[S2] https://yt-project.org/docs/dev/visualizing/volume_rendering.html#volume-rendering-components, https://yt-project.org/docs/dev/visualizing/volume_rendering.html, fetched 2026-10-29
