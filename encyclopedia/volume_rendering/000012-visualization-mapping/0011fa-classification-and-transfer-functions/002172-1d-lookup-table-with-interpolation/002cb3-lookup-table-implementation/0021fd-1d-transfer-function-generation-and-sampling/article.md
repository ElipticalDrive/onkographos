# 1D Transfer Function Generation and Sampling

Also called: 1D Transfer Function, 1D Transfer Function Texture Generation, TF1D, TFN Sampling to Texture, TFN Texture Upload

## Overview
1D Transfer Function Generation and Sampling maps scalar data values to color and opacity using a 1D lookup table (LUT) implemented as a 1D texture [S1, book][S7]. It solves the problem of visualizing scalar fields by assigning optical properties based on a single data measure, typically the voxel value [S1, book]. Unlike 3D color cube interpolation, which interpolates RGB vectors directly and can produce colors outside the defined colorbar (e.g., purple between red and blue), this method interpolates color indices or positions within a 1D colorbar, ensuring all displayed colors remain physically meaningful and contained within the defined spectrum [S2, unofficial]. It builds on [Lookup Table Implementation](5347-lookup-table-implementation.md) by defining how the LUT content is generated from control points and uploaded for GPU sampling.

## How it works
**Inputs:**
- Scalar data values (range $[s_{min}, s_{max}]$).
- Control points defining the transfer function: pairs of (scalar value, RGBA color) or separate color and alpha control points [S3, S5, unofficial].
- Optional Gaussian opacity objects [S5, unofficial].
- LUT resolution (size) [S5, unofficial].

**Outputs:**
- A 1D RGBA texture (LUT) where each entry maps a normalized scalar value to an RGBA color [S1, book][S5, unofficial].
- Sampled RGBA values for each voxel during rendering [S1, book].

**Data Structures:**
- **ControlPoint:** Contains `value` (scalar) and `color` (TFColor) [S3, unofficial].
- **TFColor:** Contains `r`, `g`, `b`, `a` (floats in $[0.0, 1.0]$) [S3, unofficial].
- **ColorControl:** Contains `position` (float) and `color` (vec3f) [S5, unofficial].
- **AlphaControl:** Contains `pos` (vec2f) where `pos.x` is value and `pos.y` is alpha [S5, unofficial].
- **GaussianObject:** Contains `mean`, `sigma`, `heightFactor`, and `alphaArray` (vector<float>) [S5, unofficial].
- **m_rgbaTable:** Vector of `vec4f` of size `resolution` [S5, unofficial].
- **m_alphaArray:** Vector of `float` of size `resolution` [S5, unofficial].
- **palette:** `std::vector<uint8_t>` of size `tfn_w * tfn_h * 4` for OpenGL upload, layout row-major RGBA [S5, unofficial].

**Steps:**

1. **Initialize Control Points:**
   - Define control points with scalar values and RGBA colors.
   - Normalize RGB components from input (0-255) to $[0.0, 1.0]$ by dividing by 255.0 [S3, unofficial].
   - Normalize alpha values to two decimal places: `alpha = floor(alpha * 100) / 100.0` [S3, unofficial].
   - Sort control points by scalar value [S5, unofficial].
   - If no control points exist, add a default point (e.g., 0.0, 0.0, 0.0, 0.0) [S5, unofficial].

2. **Build LUT (Color):**
   - Iterate $i$ from $0$ to `resolution` - 1 [S5, unofficial].
   - Compute normalized value: $v = \frac{i + 0.5}{\text{resolution}}$ (center-aligned bin) [S5, unofficial].
   - Find the interval $[p_L, p_R]$ in sorted color controls such that $p_L \le v < p_R$ using `upper_bound` [S5, unofficial].
   - Determine interpolation weight: $w = \frac{|v - p_L|}{|p_R - p_L|}$ [S5, unofficial].
   - Interpolate color: $c = \text{mix}(c_L, c_R, w)$ where $c_L$ and $c_R$ are colors of left and right control points [S5, unofficial].
   - Store in `m_rgbaTable[i]`: `m_rgbaTable[i].rgb = c` [S5, unofficial].

3. **Build LUT (Alpha):**
   - Initialize `m_alphaArray` with zeros [S5, unofficial].
   - If alpha controls exist:
     - Sort alpha controls by position [S5, unofficial].
     - Iterate $i$ from $0$ to `resolution` - 1 [S5, unofficial].
     - Compute normalized value: $v = \frac{i}{\text{resolution} - 1}$ (edge-aligned) [S5, unofficial].
     - Find interval and interpolate alpha: $\alpha = \text{mix}(\alpha_L, \alpha_R, w)$ [S5, unofficial].
     - Update `m_rgbaTable[i].a = max(m_rgbaTable[i].a, alpha)` [S5, unofficial].
   - If Gaussian objects exist:
     - For each Gaussian, compute its alpha contribution at each LUT index and update `m_rgbaTable[i].a = max(m_rgbaTable[i].a, gaussian_alpha[i])` [S5, unofficial].
   - **Note:** The misalignment between center-aligned color bins and edge-aligned alpha bins can cause slight boundary discrepancies [S5, unofficial].

4. **Upload to GPU:**
   - Convert `m_rgbaTable` to `palette` vector of `uint8_t` (multiply by 255 and clamp) [S5, unofficial].
   - Upload using `glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA8, tfn_w, tfn_h, 0, GL_RGBA, GL_UNSIGNED_BYTE, palette.data())` [S5, unofficial].

5. **Sampling during Rendering:**
   - Read scalar value $v$ from 3D data texture [S1, book].
   - Use $v$ as texture coordinate to sample 1D transfer function texture: `color = tex1d(v, tfTex)` [S1, book].
   - Multiply color components by opacity for correct rendering: `final_color = color.rgb * color.a` (opacity-weighted color) [S1, book][S7].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def build_lut(control_points, alpha_controls, gaussian_objects, resolution):
    """
    Builds a 1D LUT from control points.
    control_points: list of (value, r, g, b, a)
    alpha_controls: list of (value, alpha)
    gaussian_objects: list of GaussianObject instances
    resolution: int, size of LUT
    Returns: list of (r, g, b, a) tuples
    """
    # Normalize and sort color control points
    normalized_color_controls = []
    for cp in control_points:
        val, r, g, b, a = cp
        # Normalize RGB to [0, 1] if input is 0-255
        r, g, b = r / 255.0, g / 255.0, b / 255.0
        # Normalize alpha to 2 decimal places
        a = math.floor(a * 100) / 100.0
        normalized_color_controls.append((val, r, g, b, a))
    
    if not normalized_color_controls:
        normalized_color_controls.append((0.0, 0.0, 0.0, 0.0, 0.0))
    
    normalized_color_controls.sort(key=lambda x: x[0])
    
    # Initialize LUT
    lut = [None] * resolution
    
    # Build Color LUT
    for i in range(resolution):
        # Center-aligned bin
        v = (i + 0.5) / resolution
        
        # Find interval
        left_cp = None
        right_cp = None
        for j in range(len(normalized_color_controls) - 1):
            if normalized_color_controls[j][0] <= v < normalized_color_controls[j+1][0]:
                left_cp = normalized_color_controls[j]
                right_cp = normalized_color_controls[j+1]
                break
        
        if left_cp is None:
            # Clamp to first or last
            if v < normalized_color_controls[0][0]:
                left_cp = normalized_color_controls[0]
                right_cp = normalized_color_controls[0]
            else:
                left_cp = normalized_color_controls[-1]
                right_cp = normalized_color_controls[-1]
        
        # Interpolate color
        if left_cp == right_cp:
            r, g, b = left_cp[1], left_cp[2], left_cp[3]
        else:
            w = abs(v - left_cp[0]) / abs(right_cp[0] - left_cp[0])
            r = left_cp[1] + w * (right_cp[1] - left_cp[1])
            g = left_cp[2] + w * (right_cp[2] - left_cp[2])
            b = left_cp[3] + w * (right_cp[3] - left_cp[3])
        
        lut[i] = [r, g, b, 0.0] # Alpha initialized to 0
    
    # Build Alpha LUT
    if alpha_controls:
        normalized_alpha_controls = sorted(alpha_controls, key=lambda x: x[0])
        for i in range(resolution):
            # Edge-aligned bin for alpha
            v = i / (resolution - 1)
            
            left_ac = None
            right_ac = None
            for j in range(len(normalized_alpha_controls) - 1):
                if normalized_alpha_controls[j][0] <= v < normalized_alpha_controls[j+1][0]:
                    left_ac = normalized_alpha_controls[j]
                    right_ac = normalized_alpha_controls[j+1]
                    break
            
            if left_ac is None:
                if v < normalized_alpha_controls[0][0]:
                    left_ac = normalized_alpha_controls[0]
                    right_ac = normalized_alpha_controls[0]
                else:
                    left_ac = normalized_alpha_controls[-1]
                    right_ac = normalized_alpha_controls[-1]
            
            if left_ac == right_ac:
                alpha = left_ac[1]
            else:
                w = abs(v - left_ac[0]) / abs(right_ac[0] - left_ac[0])
                alpha = left_ac[1] + w * (right_ac[1] - left_ac[1])
            
            # Max combination
            lut[i][3] = max(lut[i][3], alpha)
    
    # Add Gaussian Opacity
    for gauss in gaussian_objects:
        gauss_alpha_array = gauss.update_alpha_array(resolution)
        for i in range(resolution):
            lut[i][3] = max(lut[i][3], gauss_alpha_array[i])
            
    return lut

def sample_tf(v, lut, s_min, s_range):
    """
    Samples the LUT for a given scalar value v.
    v: scalar value
    lut: list of (r, g, b, a)
    s_min: minimum scalar value in data
    s_range: range of scalar values (s_max - s_min)
    Returns: (r, g, b, a)
    """
    # Map v to LUT index
    index = int(((len(lut) - 1) * (v - s_min)) / s_range)
    index = max(0, min(index, len(lut) - 1)) # Clamp
    return lut[index]

def render_voxel(v, data_tex, tf_tex):
    """
    Fragment shader equivalent.
    v: scalar value from data_tex
    tf_tex: 1D transfer function texture
    Returns: final color
    """
    # Sample TF
    color_rgba = sample_tf(v, tf_tex) # Assuming tf_tex is the LUT
    r, g, b, a = color_rgba
    
    # Opacity-weighted color
    final_r = r * a
    final_g = g * a
    final_b = b * a
    return (final_r, final_g, final_b, a)
```

## Subtopics

### [Voxel Color Mapping](2970-voxel-color-mapping.md)
Maps voxel values to colors using the 1D LUT. It is the core application of the TF1D method, where each voxel's scalar value is used as an index into the LUT to retrieve its RGBA color. It is chosen when scalar values directly correspond to material properties or visual features. Link: [Voxel Color Mapping](2970-voxel-color-mapping.md).

### [Color Specification](3969-color-specification.md)
Defines colors using four components: red, green, blue, and alpha (opaqueness), each ranging from 0.0 to 1.0. This specification is used for control points and LUT entries. It is chosen when precise control over color and opacity is required. Link: [Color Specification](3969-color-specification.md).

### [Parametric Function Generation](5349-parametric-function-generation.md)
Generates the transfer function using parametric functions (e.g., Gaussians) in addition to or instead of control points. It is chosen when smooth, continuous opacity profiles are needed, such as for highlighting specific value ranges. Link: [Parametric Function Generation](5349-parametric-function-generation.md).

### Hounsfield Scale Color Mapping
Assigns colors to voxels based on attenuation coefficients (HU values) from CT scans. It is a specialized 1D TF for medical imaging, mapping HU values to colors (e.g., bone, tissue, air). It is chosen for CT volume visualization. Link: [Hounsfield Scale Color Mapping](2970-voxel-color-mapping.md) (Note: No specific article, but described in notes).

### Magnitude Color Gradient
Visualizes vector magnitude (e.g., velocity) using a 1D CUDA texture with a blue-green-red gradient. It is chosen for vector field visualization where magnitude is the key feature. Link: [Magnitude Color Gradient](2970-voxel-color-mapping.md) (Note: No specific article, but described in notes).

### 3D Color Cube Interpolation
Interpolates colors directly in 3D RGB space. It is an alternative to 1D TF but produces artifacts (e.g., purple between red and blue) and is generally avoided. It is used in older versions of Smokeview. Link: [3D Color Cube Interpolation](2970-voxel-color-mapping.md) (Note: No specific article, but described in notes).

### Gradient/Color Ramp Sampling
Samples colors from a gradient texture, often with truncation or HDR handling. It is chosen for compatibility with specific engines like Godot. Link: [Gradient/Color Ramp Sampling](2970-voxel-color-mapping.md) (Note: No specific article, but described in notes).

### Bilinear Image Sampling
Samples colors from 2D images using bilinear interpolation. It is chosen for texture mapping or image-based transfer functions. Link: [Bilinear Image Sampling](2970-voxel-color-mapping.md) (Note: No specific article, but described in notes).

### Color Blending
Linearly interpolates between two colors based on a ratio. It is a basic operation used in TF generation and rendering. Link: [Color Blending](2970-voxel-color-mapping.md) (Note: No specific article, but described in notes).

## Operating characteristics
- **Time Complexity:** LUT construction is $O(N \cdot C)$ where $N$ is resolution and $C$ is number of control points (or $O(N \log C)$ with binary search) [S5, unofficial]. Sampling is $O(1)$ per voxel [S1, book].
- **Memory Complexity:** $O(N)$ for LUT storage, where $N$ is resolution (e.g., 1024 entries) [S5, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** LUT size can be increased for higher precision, but memory usage grows linearly [S5, unofficial].
- **Strengths:** Simple, efficient, ensures colors stay within the defined colorbar [S2, unofficial].
- **Limitations:** Only uses scalar value; cannot incorporate gradient or other measures without 2D TF [S1, book]. Misalignment between color and alpha binning can cause boundary issues [S5, unofficial].
- **When to use:** When scalar values are sufficient for classification and performance is critical [S1, book].
- **When not to use:** When gradient information is needed for better feature separation [S1, book].

## Background
The method originated with the use of 1D texture lookup tables for volume rendering, as described in NVIDIA's GPU Gems [S1, book][S7]. It was further detailed in Smokeview's technical reference, which highlighted the advantages of interpolating color indices over 3D color cube interpolation to avoid artifacts [S2, unofficial]. The use of Gaussian objects for opacity profiles was introduced in later implementations [S5, unofficial]. It is widely used in medical imaging (e.g., Hounsfield scale mapping) [S8, unofficial] and vector field visualization [S9, unofficial].

## Sources
[S1] Chapter 39. Volume Rendering Techniques | NVIDIA Developer, https://developer.nvidia.com/gpugems/gpugems/part-vi-beyond-triangles/chapter-39-volume-rendering-techniques, fetched 2026-10-26 (book)
[S2] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S3] faviasono/raycastingrendering, https://github.com/faviasono/raycastingrendering, fetched 2026-10-26 (unofficial)
[S4] fraserlove/ossium, https://github.com/fraserlove/ossium, fetched 2026-10-27 (unofficial)
[S5] wilsonCernWq/TransferFunctionModule, https://github.com/wilsonCernWq/TransferFunctionModule, fetched 2026-10-27 (unofficial)
[S6] hopexn/VolumeRenderingParallel, https://github.com/hopexn/VolumeRenderingParallel, fetched 2026-10-27 (unofficial)
[S7] GPU Gems ch39, Volume rendering techniques, https://developer.nvidia.com/sites/all/modules/custom/gpugems/books/GPUGems/gpugems_ch39.html, fetched 2026-10-29
[S8] Volume Raycasting mit OpenCL, https://arxiv.org/abs/1609.01317v1, fetched 2026-10-26 (unofficial)
[S9] Realtime visualization of 3D vector fields using CUDA | Marek's blog, https://marekfiser.com/projects/realtime-visualization-of-3D-vector-fields-using-CUDA/, fetched 2026-10-28 (unofficial)
[S10] feat(renderer): draw FogVolume and FogMaterial into the volumetric fog, https://github.com/Vortiago/textscene-inspector/pull/732, fetched 2026-10-26 (unofficial)
[S11] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S12] Musta-Kissa/quadtree-raycast, https://github.com/Musta-Kissa/quadtree-raycast, fetched 2026-10-24 (unofficial)
