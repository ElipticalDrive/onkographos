# Voxel Color Mapping

Also called: Color Index Mapping [S1, unofficial]

## Overview
Voxel Color Mapping converts scalar volume data values into integer indices for a color table lookup, assigning RGB colors and opacity to each voxel [S1, unofficial]. It solves the problem of quantitative data visualization by mapping numerical values to visual appearance [S4, unofficial]. Unlike [Parametric Function Generation](5349-parametric-function-generation.md) which computes values analytically, or [Color Blending](5150-color-blending.md) which composites final pixels, this method performs a deterministic, per-voxel assignment based on a 1D Lookup Table (LUT) [S1, unofficial]. It builds on [1D Transfer Function Generation and Sampling](2689-1d-transfer-function-generation-and-sampling.md) by utilizing the generated LUT to index colors [S1, unofficial].

## How it works
**Inputs:**
- A 3D volume grid of scalar values $v_{i,j,k}$ [S4, unofficial].
- Data bounds $v_{min}$ and $v_{max}$ [S1, unofficial].
- A color table (LUT) with 256 entries [S1, unofficial].

**Outputs:**
- A 3D volume grid of RGBA values (color and opacity) for each voxel [S4, unofficial].

**Data Structures:**
- **Color Table (LUT):** An array of 256 entries [S1, unofficial].
- Each entry contains 4 components: Red, Green, Blue, and Opacity [S1, unofficial].
- Each component is scaled from 0.0 to 1.0 [S1, unofficial].
- **Voxel Data:** Input scalar values [S4, unofficial].
- Output RGBA values stored per voxel [S4, unofficial].

**Steps:**
1. **Determine Data Bounds:** Establish $v_{min}$ and $v_{max}$ [S1, unofficial]. These can be set by the user, calculated via global min/max, or determined using percentile bounds (e.g., 1st and 99th percentiles) by scanning the data and computing a histogram [S1, unofficial].
2. **Scale Data Values:** Linearly scale the original data values $v$ to lie between the minimum and maximum representable by the number of bits available for each voxel (e.g., 0 to 255 for 8-bit) [S4, unofficial].
3. **Compute Color Index:** Map the scaled value $v$ to an integer index between 0 and 255 using the following formula [S1, unofficial]:
   $$
   \text{color index} = \begin{cases} 
   0 & v < v_{min} \\
   1 + 253(v - v_{min})/(v_{max} - v_{min}) & v_{min} \leq v \leq v_{max} \\
   255 & v > v_{max}
   \end{cases}
   $$
   where $v_{min}$ and $v_{max}$ are the specified data bounds [S1, unofficial].
4. **Lookup Color and Opacity:** Use the computed index to retrieve the RGBA values from the LUT [S1, unofficial].
   $$
   \text{Opacity}[i,j,k] = \text{LUT}[\text{color index}]
   $$
   The color is similarly retrieved from the LUT entry [S4, unofficial].
5. **Assign Voxel Properties:** Assign the retrieved color and opacity to the voxel at $(i,j,k)$ [S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_color_mapping(volume_data, v_min, v_max, color_table):
    """
    volume_data: 3D array of scalar values
    v_min: float, minimum data bound
    v_max: float, maximum data bound
    color_table: list of 256 tuples, each tuple is (r, g, b, opacity) with values 0.0-1.0
    Returns: 3D array of RGBA tuples
    """
    output_volume = []
    for k in range(volume_data.shape[0]):
        row = []
        for j in range(volume_data.shape[1]):
            voxel_rgba = []
            for i in range(volume_data.shape[2]):
                v = volume_data[k, j, i]
                if v < v_min:
                    index = 0
                elif v > v_max:
                    index = 255
                else:
                    index = 1 + 253 * (v - v_min) / (v_max - v_min)
                index = int(round(index))
                rgba = color_table[index]
                voxel_rgba.append(rgba)
            row.append(voxel_rgba)
        output_volume.append(row)
    return output_volume
```

**Edge Cases and Pitfalls:**
- **Precision:** The index calculation uses linear interpolation [S1, unofficial]. Ensure floating-point precision does not cause index out-of-bounds errors [G].
- **Boundaries:** Values outside $[v_{min}, v_{max}]$ are clamped to 0 or 255 [S1, unofficial].
- **Surface Accuracy:** Straightforward mapping of value to opacity may not accurately display surfaces, particularly interfaces like air/skin or skin/muscle [S4, unofficial]. Gradient-based opacity is often needed for accurate surface visualization [S4, unofficial].

## Subtopics

### [Scalar/Component to Color Mapping](6697-scalar-component-to-color-mapping.md)
Scalar/Component to Color Mapping converts a single scalar volume value or a specific component of a surface normal into an RGB color using a 2D colormap texture. It performs a direct lookup based on a normalized input, often used in direct volume rendering pipelines. Unlike complex transfer functions that map multiple attributes to opacity and color, this method assigns a fixed color based on a single normalized scalar.

### [Palette-based Color Mapping](7283-palette-based-color-mapping.md)
Palette-based color mapping maps discrete color indices to UV coordinates on a square texture atlas, avoiding the storage of per-voxel color data in vertex buffers. It differs from [Scalar/Component to Color Mapping](6697-scalar-component-to-color-mapping.md) by using a pre-computed lookup table (palette) rather than procedural or analytical functions to determine color, and from [Voxel Color Mapping](2970-voxel-color-mapping.md) by explicitly decoupling the color definition from the vertex geometry via a shared texture resource.

### Multi-hypothesis Voxel Coloring
Multi-hypothesis Voxel Coloring identifies a set of hypotheses (possible colorings) for each voxel and narrows them down during a hypothesis removal step, carving inconsistent voxels [S5, unofficial]. This approach is used in volumetric reconstruction to refine the representation of semi-transparent or opaque objects from multiple views [S5, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels, as each voxel requires a constant-time lookup and arithmetic operation [S1, S4, unofficial].
- **Memory Complexity:** $O(N)$ for the output volume, plus $O(1)$ for the LUT (fixed size 256 entries) [S1, unofficial].
- **Limitations:** Does not account for gradients or local features, which can lead to inaccurate surface displays [S4, unofficial].
- **When to Use:** When a direct mapping of scalar values to color is sufficient [S1, unofficial].
- **When Not to Use:** When accurate surface visualization is required, especially for medical imaging with complex interfaces [S4, unofficial].

## Background
Voxel Color Mapping originated as a fundamental technique in volume visualization, particularly in tools like Smokeview for visualizing fire dynamics simulation data [S1, unofficial]. It is part of a larger classification process where data values are converted into opacity and color to indicate voxel importance [S4, unofficial].

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S2] Volumetric Render Engine - GitHub, https://github.com/mikejernil/volumetric-render-engine, fetched 2026-10-27 (unofficial)
[S3] scarlyle0/voxel-compare, https://github.com/scarlyle0/voxel-compare, fetched 2026-10-27 (unofficial)
[S4] 14 Acceleration Techniques for Volume Rendering Mark W. Jones 14 Acceleration Techniques for Volume Rendering Mark W. Jones, https://docslib.org/doc/3014834/14-acceleration-techniques-for-volume-rendering-mark-w-jones, fetched 2026-10-28 (unofficial)
[S5] Accelerated volumetric reconstruction from uncalibrated camera views, http://doras.dcu.ie/157/, fetched 2026-10-26 (unofficial)
