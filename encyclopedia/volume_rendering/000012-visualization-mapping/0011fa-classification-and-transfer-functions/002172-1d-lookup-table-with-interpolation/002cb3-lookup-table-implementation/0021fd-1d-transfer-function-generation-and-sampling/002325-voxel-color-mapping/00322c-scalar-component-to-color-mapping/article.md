# Scalar/Component to Color Mapping

Also called: Pseudocoloring, Scalar Coloring, Spectrum Color Mapping, Target Value Visualization, Violet-Red Spectrum Mapping, Visualization Target Mapping, 颜色映射

## Overview
Scalar/Component to Color Mapping is a visualization method that maps a single scalar value or a specific vector component (such as a normal's X, Y, or Z axis) to an RGB color using a 2D colormap texture [S2, unofficial]. It solves the problem of visualizing scalar magnitude or directional components by converting numerical data into perceptible color cues [S1, unofficial]. Builds on [Voxel Color Mapping](2970-voxel-color-mapping.md), this method typically employs a fixed, global linear normalization over the dataset bounds (defined by a `u_clim` range) to index into a pre-defined color table [S2, unofficial]. It differs from [Palette-based Color Mapping](7283-palette-based-color-mapping.md) by relying on continuous scalar interpolation within a texture rather than discrete palette indices [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar value $v$ or a vector component (e.g., normal vector $\mathbf{n}$).
*   **Input:** A colormap texture $T$ (2D sampler).
*   **Input:** A clim range $[c_{min}, c_{max}]$ (or `u_clim`).
*   **Input:** A target selector `visualize_target` (enum: `VALUE`, `NORMAL_X`, `NORMAL_Y`, `NORMAL_Z`).
*   **Output:** An RGBA color value.

**Data Structures:**
*   `u_clim`: A `vec2` containing the lower and upper bounds of the value range for normalization [S2, unofficial].
*   `u_colormap`: A 2D sampler texture containing the color map [S2, unofficial].
*   `visualize_target`: An enumeration determining which data component to visualize [S2, unofficial].

**Steps:**
1.  **Select Source Value:** Determine the scalar value $s$ to be mapped based on `visualize_target` [S2, unofficial].
    *   If `visualize_target` is `VALUE`, $s = v$ [S2, unofficial].
    *   If `visualize_target` is `NORMAL_X`, $s = \mathbf{n}.x$ [S2, unofficial].
    *   If `visualize_target` is `NORMAL_Y`, $s = \mathbf{n}.y$ [S2, unofficial].
    *   If `visualize_target` is `NORMAL_Z`, $s = \mathbf{n}.z$ [S2, unofficial].
2.  **Normalize:** Map $s$ to the range $[0, 1]$ [S2, unofficial].
    *   For `VALUE`: Normalize using the clim range:
        $$ s_{norm} = \frac{s - c_{min}}{c_{max} - c_{min}} $$
        [S2, unofficial].
    *   For `NORMAL_*` components: Since normal components are in $[-1, 1]$ [S2, unofficial], map to $[0, 1]$ via:
        $$ s_{norm} = \frac{s + 1.0}{2.0} $$
        [S2, unofficial].
3.  **Clamp:** Clamp $s_{norm}$ to $[0, 1]$ to handle out-of-bounds values [S2, unofficial].
4.  **Sample:** Sample the colormap texture $T$ at the normalized x-coordinate and a fixed y-coordinate (typically 0.5 for 1D colormaps stored in 2D textures):
    $$ \text{color} = T(s_{norm}, 0.5) $$
    [S2, unofficial].
5.  **Return:** Return the sampled RGB color with alpha set to 1.0 [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Enum constants for visualize_target
VALUE = 0
NORMAL_X = 1
NORMAL_Y = 2
NORMAL_Z = 3

def scalar_to_color(value, normal, visualize_target, u_clim, u_colormap):
    """
    Maps a scalar value or normal component to a color using a colormap texture.
    
    Args:
        value (float): The scalar volume value.
        normal (vec3): The normal vector at the current voxel.
        visualize_target (int): One of VALUE, NORMAL_X, NORMAL_Y, NORMAL_Z.
        u_clim (vec2): [c_min, c_max] range for normalization.
        u_colormap (sampler2D): The 2D colormap texture.
        
    Returns:
        vec4: The resulting RGBA color.
    """
    s = 0.0
    
    # Step 1: Select source value
    if visualize_target == VALUE:
        s = value
    elif visualize_target == NORMAL_X:
        s = normal.x
    elif visualize_target == NORMAL_Y:
        s = normal.y
    elif visualize_target == NORMAL_Z:
        s = normal.z
    else:
        s = value

    s_norm = 0.0

    # Step 2: Normalize
    if visualize_target == VALUE:
        # Guard against division by zero if c_min == c_max [G]
        if u_clim[1] == u_clim[0]:
            s_norm = 0.5
        else:
            s_norm = (s - u_clim[0]) / (u_clim[1] - u_clim[0])
    else:
        # Normalize normal components from [-1, 1] to [0, 1]
        s_norm = (s + 1.0) / 2.0

    # Step 3: Clamp
    s_norm = max(0.0, min(1.0, s_norm))

    # Step 4: Sample colormap
    # Assuming 1D colormap stored in 2D texture at y=0.5
    color = texture(u_colormap, vec2(s_norm, 0.5))

    # Step 5: Return with alpha 1.0
    return vec4(color.rgb, 1.0)
```

**Parameters:**
*   `u_clim`: A `vec2` containing the lower and upper bounds of the value range for normalization [S2, unofficial].
*   `visualize_target`: Enum determining whether to visualize `NORMAL_X`, `NORMAL_Y`, `NORMAL_Z`, or `VALUE` [S2, unofficial].
*   `u_colormap`: A 2D sampler texture containing the color map [S2, unofficial].
*   `cm_type`: Colormap type (e.g., from `tinycolormap::ColormapType`) [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Clim Range Equality:** If $c_{min} == c_{max}$, normalization results in division by zero [G]. A guard (e.g., defaulting to 0.5) is required.
*   **Normal Component Range:** Normal components are in $[-1, 1]$ [S2, unofficial]. Failing to shift and scale them to $[0, 1]$ before indexing into a colormap defined on $[0, 1]$ will result in incorrect colors or out-of-bounds sampling [S2, unofficial].
*   **Clamping:** Input values outside $[0, 1]$ are clamped to $[0, 1]$ before evaluation to prevent out-of-bounds or negative color components [S2, unofficial].

## Subtopics

### [Colormap Conversion](7117-colormap-conversion.md)
Colormap Conversion maps scalar data values to RGB colors by linearly scaling the input to a normalized index and indexing into a pre-defined color table. It solves the problem of visualizing scalar magnitude (e.g., temperature) through color. It applies a fixed, global linear normalization over the dataset bounds to index a pre-defined color table.

### [Scalar to Color Mapping Scale](11536-scalar-to-color-mapping-scale.md)
The type of scale used to map scalar values to the transfer function range. Options are linear or logarithmic (base 10). The default scale type is linear. If the scale is set to logarithmic but the scalar range contains zero, the color mapping defaults to linear to avoid undefined behavior.

### Lookup-free Transfer Function
Maps a normalized scalar value $t$ in $[0, 1]$ to an RGB color using piecewise polynomial and absolute-value functions without a texture lookup [S7, unofficial]. The input is clamped to $[0, 1]$ before evaluation to prevent out-of-bounds or negative color components [S7, unofficial]. The mapping formulas are:
$$ t = \text{clamp}(value, 0, 1) $$
$$ r = \lfloor 14 + t \times 230 \rfloor $$
$$ g = \lfloor 24 + t^{0.7} \times 174 \rfloor $$
$$ b = \lfloor 26 + (1 - |t - 0.6|) \times 116 \rfloor $$
[S7, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per voxel/pixel for the mapping itself, as it involves a constant number of arithmetic operations and a single texture lookup [S2, unofficial].
*   **Memory Complexity:** Depends on the size of the colormap texture [S2, unofficial].
*   **Strengths:** Simple to implement; efficient for real-time rendering; allows easy switching between scalar and component visualization [S2, unofficial].
*   **Limitations:** Linear normalization may not reveal details in data with non-uniform distributions [G].
*   **Colormap Choice:** Perceptually uniform colormaps (e.g., Viridis) are preferred over Rainbow colormaps because Rainbow colormaps suffer from 'false boundary' artifacts due to non-uniform human perception of hue intervals [S6, unofficial]. For data with positive/negative distinctions, a diverging colormap should be used [S6, unofficial]. Common schemes include Rainbow, Viridis, Diverging, Grayscale, and Hot [S6, unofficial].

## Background
Scalar/Component to Color Mapping is the most common scalar field visualization method, mapping scalar values to a color space [S6, unofficial]. It is widely used in direct volume rendering and surface visualization [S2, unofficial]. The method relies on the concept of transfer functions, where scalar values are mapped to optical properties (color and opacity) [S6, unofficial]. The use of logarithmic scales for mapping is a feature found in visualization libraries to handle data with wide dynamic ranges [S1, unofficial].

## Sources
[S1] DTIC ADA259586: Scientific Visualization of Volumetric Radar Cross Section Data, https://archive.org/details/DTIC_ADA259586, fetched 2026-10-26 (unofficial)
[S2] Masdaofdisasda/Vis1_Direct_Volume_Rendering_2022W, https://github.com/Masdaofdisasda/Vis1_Direct_Volume_Rendering_2022W, fetched 2026-10-26 (unofficial)
[S3] User's Guide - Visualization Techniques, https://theochem.mercer.edu/dxdocs/pages/usrgu063.htm, fetched 2026-10-28
[S4] GitHub - yuehaowang/volume_renderer: CUDA-based interactive volume ..., https://github.com/yuehaowang/volume_renderer, fetched 2026-10-29 (unofficial)
[S5] 4. Displaying data — ParaView Documentation 5.8.1 documentation, https://docs.paraview.org/en/v5.9.1/UsersGuide/displayingData.html, fetched 2026-10-29
[S6] 第9 章：科学可视化, http://simingchen.me/smart-textbook/ch9/index.html, fetched 2026-10-30 (unofficial)
[S7] dueyama/gray-scott-3d, https://github.com/dueyama/gray-scott-3d, fetched 2026-10-27 (unofficial)
