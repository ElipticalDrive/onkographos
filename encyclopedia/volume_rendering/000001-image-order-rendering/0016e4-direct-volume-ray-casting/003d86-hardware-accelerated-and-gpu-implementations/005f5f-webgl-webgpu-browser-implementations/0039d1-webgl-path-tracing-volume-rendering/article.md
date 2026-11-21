# WebGL Path Tracing Volume Rendering

Also called: Interactive Voxel Volume Rendering via WebGL 2.0

## Overview
An interactive voxel volume renderer using path tracing that runs in modern browsers via WebGL 2.0. It renders 3D scalar fields by casting rays from the camera through the volume, accumulating color and opacity along the ray, and applying lighting/shadowing [S1, unofficial]. It differs from standard ray casting by computing normals via central differences of opacity and applying Phong-like lighting with soft shadows, rather than relying solely on direct illumination or pre-computed lighting [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**:
    *   `volume`: `highp sampler3D` containing the scalar field.
    *   `volume_dims`: `ivec3` defining the dimensions of the volume texture.
    *   `transformed_eye`: `vec3` camera position in world space.
    *   `vray_dir`: `vec3` ray direction in world space.
    *   `dt_scale`: `float` step size scaling factor.
    *   `lightPosition`: `vec3` light source position.
    *   `windowLevel`, `windowWidth`: `float` for window/level adjustment.
    *   `rescaleSlope`, `rescaleIntercept`: `float` for linear transformation of raw values.
    *   `numberOfOpacitySamples`, `opacityMapPositions`, `opacityMapSamples`: `int`, `highp sampler2D`, `highp sampler2D` for opacity transfer function.
    *   `numberOfColorSamples`, `colorMapPositions`, `colorMapSamples`: `int`, `highp sampler2D`, `highp sampler2D` for color transfer function.
*   **Output**: `vec4` color (RGBA) for the fragment.

**Data Structures**
*   **Opacity/Color Maps**: Stored in 2D textures. The x-coordinate represents the position (control point), and the y-coordinate represents the sample value (opacity or color intensity). The number of control points is given by `numberOfOpacitySamples` or `numberOfColorSamples`.
*   **Volume**: Stored in a 3D texture (`sampler3D`).

**Algorithm Steps**

1.  **Ray-Box Intersection**:
    Compute the intersection of the ray with the volume's bounding box.
    $$
    \text{invDir} = \frac{1.0}{\text{dir}}
    $$
    $$
    t_{\text{min\_tmp}} = (\text{boxMin} - \text{orig}) \times \text{invDir}
    $$
    $$
    t_{\text{max\_tmp}} = (\text{boxMax} - \text{orig}) \times \text{invDir}
    $$
    $$
    t_{\text{min}} = \min(t_{\text{min\_tmp}}, t_{\text{max\_tmp}})
    $$
    $$
    t_{\text{max}} = \max(t_{\text{min\_tmp}}, t_{\text{max\_tmp}})
    $$
    $$
    t_0 = \max(t_{\text{min}.x, \max(t_{\text{min}.y, t_{\text{min}.z})})
    $$
    $$
    t_1 = \min(t_{\text{max}.x, \min(t_{\text{max}.y, t_{\text{max}.z})})
    $$
    If $t_0 > t_1$, the ray does not intersect the volume; discard the fragment [S1, unofficial].

2.  **Step Size Calculation**:
    Compute the step size $dt$ based on the volume dimensions and ray direction to ensure consistent sampling density relative to voxel size.
    $$
    dt_{\text{vec}} = \frac{1.0}{\text{volume\_dims} \times |\text{ray\_dir}|}
    $$
    $$
    dt = dt_{\text{scale}} \times \min(dt_{\text{vec}.x, \min(dt_{\text{vec}.y, dt_{\text{vec}.z})})
    $$
    Typical value for $dt_{\text{scale}}$ is not specified in the sources [S1, unofficial].

3.  **Ray Origin Offset**:
    Generate a random offset to reduce aliasing.
    $$
    \text{seed} = \text{wang\_hash}(\text{int}(gl\_FragCoord.x + 1024.0 \times gl\_FragCoord.y))
    $$
    $$
    \text{offset} = \frac{\text{seed}}{2147483647.0}
    $$
    $$
    p = \text{transformed\_eye} + (t_0 + \text{offset} \times dt) \times \text{ray\_dir}
    $$
    The `wang_hash` function is defined as:
    ```python
    def wang_hash(seed):
        seed = (seed ^ 61) ^ (seed >> 16)
        seed *= 9
        seed = seed ^ (seed >> 4)
        seed *= 0x27d4eb2d
        seed = seed ^ (seed >> 15)
        return seed % 2147483647
    ```
    [S1, unofficial].

4.  **Value Rescaling**:
    Apply linear transformation to raw volume values.
    $$
    \text{rescaled\_value} = \text{rescaleSlope} \times \text{raw\_value} + \text{rescaleIntercept}
    $$
    [S1, unofficial].

5.  **Transfer Function Application**:
    *   **Opacity**: Find the interval $[opacityMapPositions(i-1), opacityMapPositions(i)]$ containing the rescaled value. Interpolate between $opacityMapSamples(i-1)$ and $opacityMapSamples(i)$ using linear interpolation. Clamp the result multiplied by $\frac{\text{value} - \text{windowFloor}}{\text{windowWidth}}$ to $[0, 1]$, where $\text{windowFloor} = \text{windowLevel} - \frac{\text{windowWidth}}{2.0}$ [S1, unofficial].
    *   **Color**: Similar to opacity, find the interval in $colorMapPositions$ and interpolate between $colorMapSamples(i-1)$ and $colorMapSamples(i)$ [S1, unofficial].

6.  **Ray Marching Loop**:
    Initialize `voxelColor = vec4(0.0)`. Loop $t$ from $t_0$ to $t_1$ with step $dt$.
    Inside the loop:
    *   Sample volume value `val` at current position $p$.
    *   Get `color` and `opacity` from transfer functions applied to `val`.
    *   Compute `val_color = vec4(opacity * color, opacity)`.
    *   Apply Beer-Lambert law approximation:
        $$
        val\_color.a = 1.0 - (1.0 - val\_color.a)^{dt_{\text{scale}}}
        $$
    *   Update accumulated color:
        $$
        voxelColor.rgb += (1.0 - voxelColor.a) \times val\_color.a \times val\_color.rgb
        $$
    *   Update accumulated alpha:
        $$
        voxelColor.a += (1.0 - voxelColor.a) \times val\_color.a
        $$
    *   If $voxelColor.a \ge 0.95$, break [S1, unofficial].
    *   Advance position: $p += \text{ray\_dir} \times dt$.

7.  **Lighting Calculation**:
    *   **Normal Estimation**: Compute voxel size:
        $$
        \text{voxelSize} = \frac{1.0}{\text{volume\_dims}}
        $$
        Compute gradient using central differences of `sampleVolumeOpacity` (which applies opacity map to rescaled volume value):
        $$
        \text{gradient}.x = \frac{\text{sampleVolumeOpacity}(p + \text{vec3}(\text{voxelSize}.x, 0, 0)) - \text{sampleVolumeOpacity}(p - \text{vec3}(\text{voxelSize}.x, 0, 0))}{2.0 \times \text{voxelSize}.x}
        $$
        Similarly for $y$ and $z$. Normalize the gradient to get `voxelNormal` [S1, unofficial].
    *   **Diffuse**:
        $$
        \text{lightDirection} = \text{normalize}(\text{lightPosition} - p)
        $$
        $$
        \text{intensity} = \max(\text{dot}(\text{voxelNormal}, \text{lightDirection}), 0.0)
        $$
    *   **Specular**:
        If $\text{intensity} > 0.0$:
        $$
        h = \text{normalize}(\text{lightDirection} + \text{transformed\_eye})
        $$
        $$
        \text{intSpec} = \max(\text{dot}(h, \text{voxelNormal}), 0.0)
        $$
        $$
        \text{spec} = \text{specular} \times \text{pow}(\text{intSpec}, \text{shininess})
        $$
        With $\text{shininess} = 8.0$, $\text{specular} = 0.5$, $\text{diffuse} = 0.6$, $\text{ambient} = 0.4$ [S1, unofficial].
    *   **Shadow**:
        If $voxelColor.a \ge 0.20$:
        $$
        \text{shadow} = \text{clamp}(\text{softShadow}(p) \times 2.0 + 0.1, 0.0, 1.0)
        $$
        Else:
        $$
        \text{shadow} = 1.0
        $$
    *   **Final Light Color**:
        $$
        \text{lightColor} = \text{shadow} \times \max(\text{intensity} \times \text{diffuse} + \text{spec}, \text{ambient})
        $$
    *   Apply `lightColor` to the accumulated `voxelColor` [S1, unofficial].

8.  **Soft Shadow Calculation**:
    Sample 8 points around the voxel position $p$ in a 3x3 grid (excluding center) with spacing $\delta = \min(\text{voxelSize}.x, \min(\text{voxelSize}.y, \text{voxelSize}.z)) \times 2.0$. For each point, compute:
    $$
    \text{pointShadow} = 1.0 - \text{traceLight}(\text{position}, \text{lightPosition})
    $$
    Sum these shadows and divide by 8.0 [S1, unofficial].

9.  **Light Tracing (Shadow Ray)**:
    Cast a ray from `voxel` towards `light`.
    $$
    \text{voxelToLight} = \text{light} - \text{voxel}
    $$
    $$
    d = \text{normalize}(\text{voxelToLight})
    $$
    $$
    o = \text{voxel} - 1000.0 \times d
    $$
    Intersect ray with box to get `box`. Compute `start = length(voxel - o)`. Sample opacity along the ray from `start` to `box.y` with step $\delta = \frac{\min(\text{voxelSize}.x, \min(\text{voxelSize}.y, \text{voxelSize}.z))}{\text{samplesPerVoxel}}$ (where $\text{samplesPerVoxel} = 1.0$). Accumulate:
    $$
    \text{integrated} += \frac{\text{val}}{\text{samplesPerVoxel} \times 4.0}
    $$
    If $\text{integrated} > 0.95$, break. Return $\text{clamp}(\text{integrated}, 0.0, 1.0)$ [S1, unofficial].

10. **SRGB Conversion**:
    Convert linear color to SRGB. If $x \le 0.0031308$, return $12.92 \times x$. Else return $1.055 \times x^{1/2.4} - 0.055$ [S1, unofficial].

**Pitfalls**
*   **Ray-box intersection failure**: If $t_0 > t_1$, the ray does not intersect the volume box. The fragment is discarded [S1, unofficial].
*   **Early termination**: The ray marching loop breaks if accumulated alpha $voxelColor.a \ge 0.95$ to save computation [S1, unofficial].
*   **Shadow ray early termination**: The light tracing loop breaks if integrated opacity $\text{integrated} > 0.95$ [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of samples along the ray. The number of samples is determined by the volume dimensions, ray direction, and `dt_scale`. Shadow calculation adds a constant overhead per lit voxel.
*   **Memory Complexity**: Dominated by the storage of the 3D volume texture and the 2D transfer function textures.
*   **Performance**: Interactive performance is achieved on modern browsers via WebGL 2.0. Performance scales with resolution and volume size.
*   **Strengths**: Interactive frame rates, browser-based accessibility, soft shadows for improved depth perception.
*   **Limitations**: Approximate lighting (normal estimation from opacity gradients may be noisy), fixed step size (unless adaptive), limited to scalar volumes.
*   **When to use**: Interactive visualization of scalar fields in web browsers.
*   **When not to use**: High-fidelity offline rendering requiring accurate normal estimation from the scalar field itself, or complex volume data types (e.g., tensor fields).

## Background
The method is implemented in the repository `termijn/webgl-volumerendering` [S1, unofficial]. It demonstrates the use of WebGL 2.0 for interactive volume rendering with path tracing techniques, including lighting and shadowing, within a web browser environment.

## Sources
[S1] termijn/webgl-volumerendering, https://github.com/termijn/webgl-volumerendering, fetched 2026-10-27 (unofficial)
