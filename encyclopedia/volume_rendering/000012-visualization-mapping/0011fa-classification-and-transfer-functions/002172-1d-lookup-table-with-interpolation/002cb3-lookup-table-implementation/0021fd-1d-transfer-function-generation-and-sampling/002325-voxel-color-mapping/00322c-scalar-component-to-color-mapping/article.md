# Scalar/Component to Color Mapping

Also called: Color Mapping, Target Value Visualization, Visualization Target Mapping

## Overview
Scalar/Component to Color Mapping converts a single scalar volume value or a specific component of a surface normal into an RGB color using a 1D colormap texture [S1, S2, unofficial]. It serves as the final step in the classification pipeline, translating numerical data into visual appearance [S1, S2, unofficial]. Unlike complex transfer functions that map multiple attributes to opacity and color, this method performs a direct lookup based on a normalized input, often used for maximum intensity projection or surface visualization where opacity is fixed or handled separately [S1, S2, unofficial].

## How it works
**Inputs:**
- `value`: The scalar volume data value (float).
- `normal`: The surface normal vector (vec3), used if `visualize_target` is set to a normal component [S2, unofficial].
- `u_clim`: A `vec2` containing the lower and upper bounds of the value range for normalization [S1, unofficial].
- `u_colormap`: A 2D sampler texture containing the color map (e.g., Viridis) [S1, S2, unofficial].
- `visualize_target`: An enum determining the input source: `VALUE`, `NORMAL_X`, `NORMAL_Y`, or `NORMAL_Z` [S2, unofficial].
- `cm_type`: Colormap type identifier (e.g., from `tinycolormap::ColormapType`) [S2, unofficial].

**Data Structures:**
- `u_colormap`: 2D texture, typically 256x1 or similar width, storing RGB values. The y-coordinate is fixed at 0.5 for 1D colormaps [S1, unofficial].

**Steps:**
1. Determine the input scalar `s` based on `visualize_target`:
   - If `VALUE`: `s = value` [S2, unofficial].
   - If `NORMAL_X`: `s = normal.x()` [S2, unofficial].
   - If `NORMAL_Y`: `s = normal.y()` [S2, unofficial].
   - If `NORMAL_Z`: `s = normal.z()` [S2, unofficial].
2. Normalize `s` to the range $[0, 1]$:
   - If `visualize_target` is `VALUE`:
     - Clamp `s` to $[0, 1]$ [S2, unofficial].
     - *Note:* Source [S1, unofficial] describes normalization using `u_clim`: $s_{norm} = \frac{s - u\_clim[0]}{u\_clim[1] - u\_clim[0]}$. Source [S2, unofficial] describes clamping `VALUE` directly to $[0, 1]$. Implementers should check if `u_clim` is active; if so, use the linear normalization; otherwise, clamp [S1, S2, unofficial].
   - If `visualize_target` is a normal component:
     - Map from $[-1, 1]$ to $[0, 1]$: $s_{norm} = \frac{s + 1.0}{2.0}$ [S2, unofficial].
     - Clamp $s_{norm}$ to $[0, 1]$ [S2, unofficial].
3. Sample the colormap texture `u_colormap` at $(s_{norm}, 0.5)$ [S1, unofficial].
4. Return the sampled RGB color with alpha 1.0 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def scalar_to_color(value, normal, u_clim, u_colormap, visualize_target):
    # Determine input scalar s
    if visualize_target == VALUE:
        s = value
    elif visualize_target == NORMAL_X:
        s = normal.x
    elif visualize_target == NORMAL_Y:
        s = normal.y
    elif visualize_target == NORMAL_Z:
        s = normal.z
    else:
        raise ValueError("Unknown visualize_target")

    # Normalize s to [0, 1]
    if visualize_target == VALUE:
        # Check if clim normalization is used (S1) or direct clamp (S2)
        # Assuming clim is provided and active for VALUE
        if u_clim is not None:
            denom = u_clim[1] - u_clim[0]
            if denom == 0:
                s_norm = 0.0
            else:
                s_norm = (s - u_clim[0]) / denom
            # Clamp to [0, 1] after normalization
            s_norm = max(0.0, min(1.0, s_norm))
        else:
            # Direct clamp as per S2
            s_norm = max(0.0, min(1.0, s))
    else:
        # Normal components are in [-1, 1], map to [0, 1]
        s_norm = (s + 1.0) / 2.0
        s_norm = max(0.0, min(1.0, s_norm))

    # Sample colormap
    # u_colormap is a 2D texture, sample at x=s_norm, y=0.5
    color = texture(u_colormap, vec2(s_norm, 0.5))
    
    # Return RGB with alpha 1.0
    return vec4(color.rgb, 1.0)
```

**Parameters:**
- `u_clim`: `vec2` for value range normalization [S1, unofficial].
- `visualize_target`: Enum for input selection [S2, unofficial].
- `cm_type`: Colormap type identifier [S2, unofficial].

**Edge Cases:**
- Division by zero in `u_clim` normalization if `u_clim[0] == u_clim[1]` [G].
- Normal components outside $[-1, 1]$ due to numerical error; clamping handles this [S2, unofficial].
- Degenerate input values (NaN, Inf); clamping or explicit checks may be needed [G].

## Subtopics

### Maximum Intensity Projection (MIP) Color Mapping
Uses the highest sampled value along the ray as input to this mapping. Sets it apart by using ray-cast aggregation rather than voxel value. Choose when visualizing dense structures like bones or vessels. [MIP](2969-maximum-intensity-projection.md)

### Surface Visualization Color Mapping
Uses surface normals as input. Sets it apart by visualizing orientation rather than density. Choose when visualizing surfaces with directional properties. [Surface Vis](2971-surface-visualization.md)

## Operating characteristics
- **Time Complexity**: $O(1)$ per voxel/sample, as it involves a single texture lookup and basic arithmetic [S1, S2, unofficial].
- **Memory Complexity**: $O(1)$ additional memory, plus the colormap texture size [S1, S2, unofficial].
- **Performance**: Very fast, suitable for real-time rendering [S1, S2, unofficial].
- **Strengths**: Simple, efficient, easy to implement [S1, S2, unofficial].
- **Limitations**: Limited to single-attribute visualization; cannot encode multiple properties like opacity and color independently [S1, S2, unofficial].
- **When to use**: For MIP, surface orientation visualization, or simple scalar visualization [S1, S2, unofficial].
- **When not to use**: When complex classification requiring multiple attributes is needed [S1, S2, unofficial].

## Background
Originating from direct volume rendering implementations, this method is commonly used in educational and open-source projects for visualizing 3D textures [S1, S2, unofficial]. It is often paired with colormaps like Viridis for perceptual uniformity [S1, unofficial].

## Sources
[S1] Masdaofdisasda/Vis1_Direct_Volume_Rendering_2022W, https://github.com/Masdaofdisasda/Vis1_Direct_Volume_Rendering_2022W, fetched 2026-10-26 (unofficial)
[S2] GitHub - yuehaowang/volume_renderer: CUDA-based interactive volume ..., https://github.com/yuehaowang/volume_renderer, fetched 2026-10-29 (unofficial)
