# Voxel Color Mapping

Also called: Color Index Mapping

## Overview
Voxel Color Mapping is a visualization method that maps a single scalar value or a specific vector component to an RGB color using a 1D colormap [S1, S2, unofficial]. It solves the problem of translating abstract numerical data into perceptible visual information, serving as the core color assignment step in direct volume rendering and surface visualization [S3, unofficial]. Unlike Palette-based Color Mapping, which assigns colors from a discrete set of predefined palettes often for categorical data, this method uses a continuous 1D texture (colormap) to provide smooth gradients, allowing for the representation of continuous magnitude or directional components [S1, S2, unofficial]. It builds on [1D Transfer Function Generation and Sampling](2689-1d-transfer-function-generation-and-sampling.md) by implementing the actual lookup and sampling of the generated transfer function [S1, S2, unofficial].

## How it works
**Inputs:**
- Scalar voxel data $V$ (density, pressure, etc.) with minimum value $v_{min}$ and maximum value $v_{max}$ [S1, S4, unofficial].
- A 1D Transfer Function (colormap) $T$ of size $N=256$ entries [S1, S2, S4, unofficial]. Each entry $T[i]$ contains 4 components: Red, Green, Blue, and Opacity, scaled from 0.0 to 1.0 [S1, unofficial].
- Voxel coordinates $(i, j, k)$ [S4, unofficial].

**Outputs:**
- RGBA color value for each voxel, used for rendering [S4, unofficial].

**Data Structures:**
- **Color Table (LUT):** An array of 256 entries [S1, unofficial]. Each entry is a struct or packed value containing 4 normalized floats (or 8-bit integers mapped to 0.0–1.0) representing $(R, G, B, \alpha)$ [S1, unofficial].
- **Volume Texture:** A 3D texture (e.g., `GL_TEXTURE_3D`) storing the scalar data, typically in a single-channel format like `GL_R8` (unsigned byte) [S2, unofficial].
- **Colormap Texture:** A 1D texture (e.g., `GL_TEXTURE_1D`) storing the color table, typically in `GL_RGBA8` format [S2, unofficial].

**Steps:**

1.  **Determine Data Bounds:** Identify $v_{min}$ and $v_{max}$ for the dataset [S1, unofficial]. This can be done by user specification, global min/max, or percentile-based histogram analysis (e.g., 1st and 99th percentiles) [S1, unofficial].
2.  **Scale Data to Index:** Convert the raw scalar value $v$ of a voxel to an integer color index $c$ between 0 and 255 using linear scaling and clamping [S1, S4, unofficial]:
    $$
    c = \begin{cases} 
    0 & v < v_{min} \\
    1 + 253 \frac{v - v_{min}}{v_{max} - v_{min}} & v_{min} \leq v \leq v_{max} \\
    255 & v > v_{max}
    \end{cases}
    $$
    [S1, unofficial]
3.  **Compute Opacity:** Normalize the scaled values to lie between 0 (transparent) and 1 (opaque) [S4, unofficial]. Straightforward value-to-opacity mapping may fail to display surfaces accurately; gradient-based opacity is often needed for interfaces [S4, unofficial].
4.  **Lookup Color:** Use the index $c$ to retrieve the RGBA color from the 1D colormap texture $T[c]$ [S1, S4, unofficial].
5.  **Render:** Combine the retrieved color and opacity with other voxels during volume rendering via front-to-back compositing [S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def map_color(v, v_min, v_max, colormap):
    """Maps a scalar value to an RGBA tuple using the 256-entry LUT [S1, unofficial]."""
    if v < v_min:
        return colormap[0]
    if v > v_max:
        return colormap[255]
    # Linear scaling formula from [S1, unofficial]
    idx = 1 + 253 * (v - v_min) / (v_max - v_min)
    idx = int(round(idx))
    idx = max(0, min(255, idx))
    return colormap[idx]

def ray_march_volume(origin, direction, step, volume_bounds, volume_sampler, colormap, v_min, v_max):
    """
    Renders a volume by marching along a ray, sampling scalar data, 
    mapping to color via LUT, and compositing front-to-back [S2, S4, unofficial].
    
    Args:
        origin: (x, y, z) ray start position.
        direction: (x, y, z) normalized ray direction.
        step: Distance between samples.
        volume_bounds: (min_x, max_x, min_y, max_y, min_z, max_z).
        volume_sampler: Function(pos) -> float returning normalized scalar in [0, 1].
        colormap: List of 256 (r, g, b, a) tuples in [0.0, 1.0].
        v_min, v_max: Data bounds for color mapping.
    Returns:
        (r, g, b, a) final pixel color.
    """
    alpha_acc = 0.0
    color_acc = (0.0, 0.0, 0.0)
    t = 0.0
    
    while t < 1000.0:  # Max ray distance
        pos = (origin[0] + direction[0] * t, 
               origin[1] + direction[1] * t, 
               origin[2] + direction[2] * t)
        
        # Check if position is inside volume bounds
        if not (volume_bounds[0] <= pos[0] <= volume_bounds[1] and
                volume_bounds[2] <= pos[1] <= volume_bounds[3] and
                volume_bounds[4] <= pos[2] <= volume_bounds[5]):
            break
            
        # Sample scalar value from volume texture [S2, unofficial]
        v = volume_sampler(pos)
        
        # Map scalar to RGBA color using LUT [S1, unofficial]
        r, g, b, a = map_color(v, v_min, v_max, colormap)
        
        # Front-to-back compositing [S4, unofficial]
        color_acc = (
            color_acc[0] + r * a * (1.0 - alpha_acc),
            color_acc[1] + g * a * (1.0 - alpha_acc),
            color_acc[2] + b * a * (1.0 - alpha_acc)
        )
        alpha_acc += a * (1.0 - alpha_acc)
        
        if alpha_acc >= 0.99:
            break
        t += step
        
    return color_acc + (alpha_acc,)
```

**Edge Cases and Pitfalls:**
- **Precision:** Values are scaled linearly between the maximum and minimum representable by the number of bits available for each voxel [S4, unofficial].
- **Boundary Conditions:** Values outside $[v_{min}, v_{max}]$ are clamped to the first or last entry of the colormap, which may hide outliers or create artifacts if bounds are not chosen carefully (e.g., using percentiles instead of global min/max) [S1, unofficial].
- **Surface Display:** Straightforward value-to-opacity mapping may not accurately display surfaces; gradient-based opacity is needed for interfaces like air/skin or skin/muscle [S4, unofficial].

## Subtopics

### [Scalar/Component to Color Mapping](6697-scalar-component-to-color-mapping.md)
Scalar/Component to Color Mapping is a visualization method that maps a single scalar value or a specific vector component (such as a normal's X, Y, or Z axis) to an RGB color using a 2D colormap texture. It solves the problem of visualizing scalar magnitude or directional components by converting numerical data into perceptible color cues. Unlike general transfer functions which may map multiple attributes or use piecewise definitions, this method typically employs a fixed, global linear normalization over the entire dataset bounds (defined by a clim range) to index into a pre-defined color table. It differs from Palette-based Color Mapping by relying on continuous scalar interpolation within a texture rather than discrete palette indices.

### [Palette-based Color Mapping](7283-palette-based-color-mapping.md)
Palette-based color mapping maps discrete color indices to UV coordinates on a square texture atlas to avoid storing per-voxel color data in vertex buffers. It differs from [Scalar/Component to Color Mapping](6697-scalar-component-to-color-mapping.md) by using a pre-computed lookup table rather than procedural functions, and from [Voxel Color Mapping](2970-voxel-color-mapping.md) by decoupling color definition from vertex geometry via a shared texture resource.

### Multi-hypothesis Voxel Coloring
Multi-hypothesis Voxel Coloring is an extension where a set of hypotheses (possible colorings) is identified for each voxel [S5, unofficial]. The algorithm narrows down these hypotheses during a hypothesis removal step, carving inconsistent voxels [S5, unofficial]. This approach is used in volumetric reconstruction to handle ambiguity in voxel coloring from multiple views [S5, unofficial].

### Manual Coloring
Manual Coloring is a straightforward approach where the user manually colors voxels, typically assigning a single uniform color to the object and floor from a palette [S6]. The user selects colors from a palette and assigns them to the objects in the scene, similar to a bucket fill tool [S6]. The network then shades the scene based on light position and generates global illumination effects [S6].

## Operating characteristics
- **Time Complexity:** $O(1)$ per voxel for lookup, assuming a fixed-size LUT (256 entries) [S1, unofficial]. The overall complexity is $O(N)$ where $N$ is the number of voxels [S4, unofficial].
- **Memory Complexity:** $O(N)$ for the volume data and $O(1)$ for the colormap (256 entries) [S1, S4, unofficial].
- **Performance:** Fast due to simple table lookup and interpolation [S1, unofficial]. Using 1D texture maps enables more detail to be obtained from the visualization due to hardware-accelerated color interpolations between grid points [S1, unofficial].
- **Strengths:** Efficient, supports smooth gradients [S1, unofficial].
- **Limitations:** Straightforward value-to-opacity mapping may not accurately display surfaces; gradient-based opacity is needed for interfaces [S4, unofficial].

## Background
Voxel Color Mapping originated as a basic technique for visualizing quantitative data in fire dynamics simulation tools like Smokeview [S1, unofficial]. It has been widely adopted in direct volume rendering and medical imaging, where classification processes convert original data values into opacity and color values indicating voxel importance [S4, unofficial]. The use of 1D texture maps for color lookup became standard in OpenGL-based renderers to leverage hardware interpolation [S1, S2, unofficial].

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S2] Volumetric Render Engine - GitHub, https://github.com/mikejernil/volumetric-render-engine, fetched 2026-10-27 (unofficial)
[S3] scarlyle0/voxel-compare, https://github.com/scarlyle0/voxel-compare, fetched 2026-10-27 (unofficial)
[S4] 14 Acceleration Techniques for Volume Rendering Mark W. Jones 14 Acceleration Techniques for Volume Rendering Mark W. Jones, https://docslib.org/doc/3014834/14-acceleration-techniques-for-volume-rendering-mark-w-jones, fetched 2026-10-28 (unofficial)
[S5] Accelerated volumetric reconstruction from uncalibrated camera views, http://doras.dcu.ie/157/, fetched 2026-10-26 (unofficial)
[S6] Neural Voxel Renderer: Learning an Accurate and Controllable Rendering Tool, https://doi.org/10.1109/cvpr42600.2020.00546, fetched 2026-10-26
