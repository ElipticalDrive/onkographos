# Saliency-based Opacity Modulation

Also called: Opacity Modulation via Spatial Importance, Saliency-guided Enhancement

## Overview
Saliency-based Opacity Modulation is an automated technique that enhances volume visualization by modifying voxel opacity based on computed saliency values to guide viewer attention [S1]. It solves occlusion and visual clutter by making less important regions transparent while preserving or enhancing the visibility of salient parts [S1][S2, unofficial]. Unlike static transfer functions, this method is data-driven, relying on spatial importance maps to determine which features deserve visual prominence [S1][S2, unofficial]. It builds on the [Importance-Driven Visualization Model](8438-importance-driven-visualization-model.md) by implementing the compositing stage where importance values directly modulate the optical density of samples [S1][S2, unofficial].

## How it works
The method operates by computing a spatial importance map for the volume data and using these values to scale the original opacity of each voxel during rendering [S1][S2, unofficial].

**Inputs and Outputs:**
*   **Input:** Original volume dataset $V$, original opacity transfer function $A(z)$, spatial importance map $\Delta D(z)$, and modulation parameters (thresholds $t_l, t_h$ and factors $k_l, k_h$) [S2, unofficial]. S1 also supports a single binary threshold parameter [S1].
*   **Output:** Rendered image with modulated opacity $A'(z)$ [S2, unofficial].

**Data Structures:**
*   **Spatial Importance Map:** A 3D grid of the same dimensions as the volume data, where each voxel $z$ stores a scalar value $\Delta D(z)$ representing local intensity variation [S2, unofficial]. The absolute value $|\Delta D(z)|$ is normalized to the range $[0, 1]$ [S2, unofficial].

**Parameters:**
*   $t_l$: Low threshold for absolute spatial importance [S2, unofficial].
*   $t_h$: High threshold for absolute spatial importance [S2, unofficial].
*   $k_l$: Factor regulating modulation for importance $< t_l$ [S2, unofficial].
*   $k_h$: Factor regulating modulation for importance $> t_h$ [S2, unofficial].
*   S1 uses a single saliency threshold of $0.2$ to separate transparent voxels from preserved/enhanced ones [S1].

**Steps:**
1.  **Compute Spatial Importance:** Calculate the spatial importance map $\Delta D(z)$ for the volume, typically using an unsharp mask filter [G].
2.  **Normalize:** Normalize $|\Delta D(z)|$ to the range $[0, 1]$ [S2, unofficial].
3.  **Modulate Opacity:** For each voxel $z$, compute the new opacity $A'(z)$ based on the original opacity $A(z)$ and the importance value $|\Delta D(z)|$ using the piecewise function from S2 [S2, unofficial]:
    $$
    A'(z) = 
    \begin{cases} 
    A(z) \cdot k_l \cdot |\Delta D(z)| & \text{if } |\Delta D(z)| < t_l \\
    A(z) \cdot k_h \cdot |\Delta D(z)| & \text{if } |\Delta D(z)| > t_h \\
    A(z) & \text{otherwise}
    \end{cases}
    $$
    Alternatively, a simpler binary thresholding approach can be used [S1]:
    *   If $|\Delta D(z)| < 0.2$, set $A'(z) = 0$ (transparent) [S1].
    *   If $|\Delta D(z)| \ge 0.2$, set $A'(z) = A(z)$ (preserve) or $A'(z) = 2 \cdot A(z)$ (enhance) [S1].
4.  **Render:** Perform standard volume raycasting using the modulated opacity values $A'(z)$ [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def clamp(val, min_val, max_val):
    return max(min_val, min(max_val, val))

def box_blur_3d(volume, radius=1):
    dims_x = len(volume)
    dims_y = len(volume[0])
    dims_z = len(volume[0][0])
    result = [[[0.0 for _ in range(dims_z)] for _ in range(dims_y)] for _ in range(dims_x)]
    for x in range(dims_x):
        for y in range(dims_y):
            for z in range(dims_z):
                sum_val = 0.0
                count = 0
                for dx in range(-radius, radius + 1):
                    for dy in range(-radius, radius + 1):
                        for dz in range(-radius, radius + 1):
                            nx = clamp(x + dx, 0, dims_x - 1)
                            ny = clamp(y + dy, 0, dims_y - 1)
                            nz = clamp(z + dz, 0, dims_z - 1)
                            sum_val += volume[nx][ny][nz]
                            count += 1
                result[x][y][z] = sum_val / count
    return result

def subtract_arrays(a, b):
    dims_x = len(a)
    dims_y = len(a[0])
    dims_z = len(a[0][0])
    return [[[a[i][j][k] - b[i][j][k] for k in range(dims_z)] for j in range(dims_y)] for i in range(dims_x)]

def absolute_values(arr):
    dims_x = len(arr)
    dims_y = len(arr[0])
    dims_z = len(arr[0][0])
    return [[[abs(arr[i][j][k]) for k in range(dims_z)] for j in range(dims_y)] for i in range(dims_x)]

def max_element(arr):
    m = arr[0][0][0]
    for i in range(len(arr)):
        for j in range(len(arr[0])):
            for k in range(len(arr[0][0])):
                if arr[i][j][k] > m:
                    m = arr[i][j][k]
    return m

def divide_arrays(arr, scalar):
    dims_x = len(arr)
    dims_y = len(arr[0])
    dims_z = len(arr[0][0])
    return [[[arr[i][j][k] / scalar for k in range(dims_z)] for j in range(dims_y)] for i in range(dims_x)]

def create_zeros_like(arr):
    dims_x = len(arr)
    dims_y = len(arr[0])
    dims_z = len(arr[0][0])
    return [[[0.0 for _ in range(dims_z)] for _ in range(dims_y)] for _ in range(dims_x)]

def compute_modulated_opacity(volume_data, original_opacity, t_l, t_h, k_l, k_h):
    """
    Computes modulated opacity based on spatial importance.
    
    Args:
        volume_data: 3D list of intensity values.
        original_opacity: 3D list of original opacity values.
        t_l: Low threshold for importance.
        t_h: High threshold for importance.
        k_l: Factor for low importance.
        k_h: Factor for high importance.
        
    Returns:
        modulated_opacity: 3D list of new opacity values.
    """
    # Step 1: Compute spatial importance (unsharp masking)
    blurred_volume = box_blur_3d(volume_data, radius=1)
    diff = subtract_arrays(volume_data, blurred_volume)
    abs_diff = absolute_values(diff)
    max_val = max_element(abs_diff)
    if max_val > 0:
        importance_map = divide_arrays(abs_diff, max_val)
    else:
        importance_map = abs_diff

    # Step 2: Modulate opacity
    dims_x = len(volume_data)
    dims_y = len(volume_data[0])
    dims_z = len(volume_data[0][0])
    modulated_opacity = create_zeros_like(volume_data)

    for x in range(dims_x):
        for y in range(dims_y):
            for z in range(dims_z):
                val = importance_map[x][y][z]
                if val < t_l:
                    modulated_opacity[x][y][z] = original_opacity[x][y][z] * k_l * val
                elif val > t_h:
                    modulated_opacity[x][y][z] = original_opacity[x][y][z] * k_h * val
                else:
                    modulated_opacity[x][y][z] = original_opacity[x][y][z]
    return modulated_opacity
```

## Subtopics

### [Radial Opacity Change](7140-radial-opacity-change.md)
A dynamic focus+context technique that modulates voxel opacity based on the distance from a central reference point, inspired by the angular velocity of a viscous fluid rotating around its center. It solves the problem of occlusion in volume rendering by making voxels further from the center more transparent, thereby revealing inner structures during rotation or zooming interactions. Unlike static saliency methods, it adapts opacity in real-time to user interaction, specifically distinguishing between global rotation (center-based) and local zooming (focus-based). See [Radial Opacity Change](7140-radial-opacity-change.md).

### [Spatial Importance Map via Unsharp Masking](7928-spatial-importance-map-via-unsharp-masking.md)
This method computes a 3D spatial importance map by subtracting the original volume intensity values from a low-pass filtered (blurred) version of itself, effectively creating an unsharp mask. It captures salient boundary and separability information to modulate color and opacity in illustrative volume rendering, distinguishing itself from radiance-based saliency methods by relying solely on input intensity, which allows it to be computed as a preprocessing step independent of lighting and transfer functions. See [Spatial Importance Map via Unsharp Masking](7928-spatial-importance-map-via-unsharp-masking.md).

### [Sparseness-based Color and Opacity Modulation](7932-sparseness-based-color-and-opacity-modulation.md)
A technique to modify visual prominence by controlling optical properties (color and opacity) based on the level of sparseness. It sets itself apart from siblings like [Spatial Importance Map via Unsharp Masking](7928-spatial-importance-map-via-unsharp-masking.md) by directly altering the saturation and transparency of ray samples rather than just the opacity or spatial weighting, ensuring that sparse objects recede visually while dense objects remain prominent. See [Sparseness-based Color and Opacity Modulation](7932-sparseness-based-color-and-opacity-modulation.md).

### [Smart Visibility](8437-smart-visibility.md)
Smart Visibility is a category of illustrative visualization techniques that extends traditional light-propagation-based visibility by incorporating high-level abstractions of human cognition, such as object relevance and viewer pre-knowledge. Unlike standard volume rendering which relies solely on optical depth and occlusion, Smart Visibility explicitly handles semantic importance to ensure relevant data remains visible even when geometrically occluded by less important structures. See [Smart Visibility](8437-smart-visibility.md).

### Saliency-based Color Modulation
Modulation of voxel color based on the spatial importance map to emphasize boundaries, using either raw importance values or absolute importance values [S2, unofficial]. The color $I'$ is computed as $I' = I + \Delta D \cdot \lambda$ for raw values or $I' = I + |\Delta D| \cdot \lambda$ for absolute values, where $I$ is the original color, $\Delta D$ is the importance value, and $\lambda$ is a modulation factor [S2, unofficial]. Typical $\lambda$ values are $-10, 0, 10, 20$ for raw values and $-5, 0, 5, 10$ for absolute values [S2, unofficial]. A pitfall is that with raw values and positive $\lambda$, the image becomes darker because most importance values are negative; with absolute values, darkening and brightening effects are reversed because all values are positive [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1][S2, unofficial].
*   **Memory Complexity:** Not given in the sources [S1][S2, unofficial].
*   **Measured Performance:** In [S2, unofficial], the method was applied to CT head and body models. The modulation allows for interactive adjustment of parameters like $k_l$ and $k_h$ [S2, unofficial].
*   **Strengths:** Effectively emphasizes boundaries and structural details [S1][S2, unofficial].
*   **Limitations:** Relies on the quality of the spatial importance map [S1][S2, unofficial].
*   **When to use:** Not given in the sources [S1][S2, unofficial].
*   **When not to use:** Not given in the sources [S1][S2, unofficial].

## Background
The concept of saliency-based modulation in volume rendering was introduced in the context of obscurance-based rendering frameworks [S1]. The specific implementation using spatial importance maps derived from intensity filtering was proposed by Ruiz, Boada, Feixas, and Sbert in 2010 [S2, unofficial]. The work builds on earlier ideas of importance-driven visualization and interactive volume illustration [S1][S2, unofficial].

## Sources
[S1] Obscurance-based Volume Rendering Framework, https://doi.org/10.2312/vg/vg-pbg08/113-120, fetched 2026-10-26
[S2] Interactive Volume Illustration Using Intensity Filtering, https://doi.org/10.2312/compaesth/compaesth10/051-058, fetched 2026-10-26 (unofficial)
