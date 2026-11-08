# 1D Transfer Function Generation and Sampling

Also called: 1D Transfer Function, TF1D, 1D Lookup Table with Interpolation, Voxel Color Mapping

## Overview
1D Transfer Function Generation and Sampling maps scalar volume data values to RGBA optical properties (color and opacity) using a one-dimensional lookup table (LUT) implemented as a 1D texture [S1, book][S9]. It solves the problem of quantitative data visualization by assigning specific visual appearances to numerical values, such as density or temperature [S1, book][S3, unofficial]. Unlike [Parametric Function Generation](5349-parametric-function-generation.md) which computes values analytically, or [Color Blending](5150-color-blending.md) which composites final pixels, this method performs a deterministic, per-voxel assignment based on a generated LUT [S1, book][S3, unofficial]. It builds on [Lookup Table Implementation](5347-lookup-table-implementation.md) by utilizing the generated LUT to index colors [S1, book]. The key distinction from naive 3D color interpolation is that it interpolates indices or values within a defined colorbar to ensure all displayed colors remain physically meaningful and within the defined spectrum [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of control points defining the transfer function, where each point consists of a scalar value (position) and an RGBA color vector. Optional inputs include Gaussian opacity objects or separate alpha control points. The volume data range ($x_{min}, x_{max}$) is used for normalization [S4, S7, S8, unofficial].
*   **Outputs:** A 1D texture (LUT) of size $N$ (typically a power of two, e.g., 256, 1024, 8192) containing pre-computed RGBA values, ready for GPU sampling [S6, S7, S8, unofficial].

**Data Structures:**
*   **ControlPoint:** Contains `value` (float/int) and `color` (TFColor). `TFColor` has fields `r`, `g`, `b`, `a` (all floats in $[0, 1]$) [S4, unofficial].
*   **LUT Array:** A vector or texture of size `resolution` (e.g., 1024). Each entry is a `vec4f` (RGBA) [S7, unofficial].
*   **Alpha Array:** A vector of size `resolution` storing opacity values, often combined with color data [S7, unofficial].

**Steps:**

1.  **Initialize Control Points:**
    *   If no control points are defined, insert a default point (e.g., position 0.0, color 0.0, 0.0, 0.0, 0.0) [S7, unofficial].
    *   Sort control points by their scalar `value` (or position) in ascending order [S4, S7, unofficial].
    *   Normalize input RGB components from $[0, 255]$ to $[0.0, 1.0]$ if necessary [S7, unofficial].
    *   Round alpha values to two decimal places for precision consistency: $a = \lfloor a \times 100 \rfloor / 100.0$ [S4, unofficial].

2.  **Build LUT (Color and Alpha Interpolation):**
    *   Let $N$ be the resolution of the LUT.
    *   Iterate through each index $i$ from $0$ to $N-1$.
    *   Compute the normalized scalar value $v$ corresponding to index $i$. Two common binning strategies exist:
        *   **Center-aligned:** $v = \frac{i + 0.5}{N}$ [S7, unofficial].
        *   **Edge-aligned:** $v = \frac{i}{N - 1}$ [S7, unofficial][S14].
        *   *Note:* Inconsistent binning between color and alpha updates can cause misalignment [S7, unofficial].
    *   Find the interval $[p_L, p_R]$ in the sorted control points such that $p_L \le v < p_R$. Use `upper_bound` to find the first control point with position $> v$; let this be the right point, and the previous one be the left point [S7, unofficial].
    *   **Boundary Handling:**
        *   If $v \le p_L$ (or no left point), use the color/alpha of the first control point [S7, unofficial].
        *   If $v \ge p_R$ (or no right point), use the color/alpha of the last control point [S7, unofficial].
    *   **Interpolation:**
        *   Calculate the interpolation weight $w$:
            $$ w = \frac{|v - p_L|}{|p_R - p_L|} $$
        *   Interpolate color components:
            $$ c = \text{mix}(c_L, c_R, w) $$
            where $c_L$ and $c_R$ are the RGBA vectors of the left and right control points [S7, unofficial].
        *   Interpolate alpha similarly if using separate alpha controls [S7, unofficial].
    *   **Gaussian Opacity:** If Gaussian objects are present, compute their contribution to opacity at index $i$ and update the LUT alpha: $m\_rgbaTable[i].w = \max(m\_rgbaTable[i].w, \text{gaussian\_alpha}[i])$ [S7, unofficial]. This uses maximum compositing, not multiplication [S7, unofficial].
    *   Store the resulting RGBA vector in `m_rgbaTable[i]` [S7, unofficial].

3.  **Upload to GPU:**
    *   Create a 1D texture (or 2D texture with height 1) with format `RGBAFloat` or `RGBA8` [S3, S7, unofficial].
    *   Set wrap mode to `Clamp` and filter mode to `Bilinear` (linear interpolation) [S3, S7, unofficial].
    *   Upload the LUT array data to the texture [S7, unofficial].

4.  **Sampling (Runtime):**
    *   For each voxel/sample, read the scalar value $s$.
    *   Normalize $s$ to $[0, 1]$ using volume min/max: $v = \frac{s - x_{min}}{x_{max} - x_{min}}$ [S3, S4, unofficial].
    *   Clamp $v$ to $[0, 1]$ to prevent out-of-bounds access [S3, S7, unofficial].
    *   Sample the 1D texture at coordinate $v$ to retrieve the RGBA color [S1, book][S9].
    *   **Opacity-Weighted Color:** Multiply the retrieved RGB components by the alpha component for correct rendering: $C_{final} = C_{rgb} \times \alpha$ [S1, book][S9].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def build_lut(control_points, resolution=1024):
    """
    Builds a 1D LUT from sorted control points.
    control_points: list of dicts with keys 'value' (float), 'color' (list of 4 floats [r,g,b,a]).
    resolution: int, size of the LUT.
    Returns: list of vec4f (RGBA) for the LUT.
    """
    if not control_points:
        # Default empty LUT
        return [[0.0, 0.0, 0.0, 0.0] for _ in range(resolution)]

    # Sort by value
    control_points.sort(key=lambda p: p['value'])
    n_controls = len(control_points)
    lut = [None] * resolution

    for i in range(resolution):
        # Center-aligned binning
        v = (i + 0.5) / resolution
        
        # Find interval
        # upper_bound returns first element > v
        left_idx = -1
        right_idx = n_controls
        
        for k in range(n_controls):
            if control_points[k]['value'] > v:
                right_idx = k
                left_idx = k - 1
                break
        
        # Boundary handling
        if right_idx == 0:
            # v is less than or equal to the first control point
            color = control_points[0]['color']
        elif left_idx == -1:
            # v is greater than or equal to the last control point
            color = control_points[-1]['color']
        else:
            # Interpolate
            p_left = control_points[left_idx]
            p_right = control_points[right_idx]
            
            # Avoid division by zero if duplicate values exist
            denom = p_right['value'] - p_left['value']
            if denom == 0:
                color = p_left['color']
            else:
                w = (v - p_left['value']) / denom
                # Linear interpolation for each component
                color = [
                    p_left['color'][j] + w * (p_right['color'][j] - p_left['color'][j])
                    for j in range(4)
                ]
        
        # Alpha normalization (optional, per source [S4, unofficial])
        # color[3] = math.floor(color[3] * 100) / 100.0
        
        lut[i] = color
        
    return lut

def sample_lut(lut, scalar_value, s_min, s_max):
    """
    Samples the LUT for a given scalar value.
    """
    if s_max == s_min:
        return lut[0]
    
    # Normalize
    v = (scalar_value - s_min) / (s_max - s_min)
    
    # Clamp to [0, 1]
    v = max(0.0, min(1.0, v))
    
    # Map to LUT index
    # Using edge-aligned mapping for sampling consistency with some implementations
    # or center-aligned if the LUT was built that way. 
    # Here we assume the LUT was built with center-aligned bins, so we sample linearly.
    # Hardware bilinear filtering handles the interpolation if uploaded as texture.
    # If manual sampling:
    index_float = v * (len(lut) - 1)
    idx = int(math.floor(index_float))
    w = index_float - idx
    
    if idx >= len(lut) - 1:
        return lut[-1]
    
    c1 = lut[idx]
    c2 = lut[idx + 1]
    
    result = [
        c1[j] + w * (c2[j] - c1[j])
        for j in range(4)
    ]
    
    # Opacity-weighted color
    alpha = result[3]
    result[0] *= alpha
    result[1] *= alpha
    result[2] *= alpha
    
    return result
```

## Subtopics

### Hounsfield Scale Color Mapping
A method to assign color values to voxels based on attenuation coefficients (e.g., from CT scans) using the Hounsfield scale [S10, unofficial]. It solves the problem of visualizing medical imaging data by mapping physical attenuation properties to colors [S10, unofficial]. Unlike generic 1D TFs, it uses a specific physical formula: $HU = \frac{\mu_{tissue} - \mu_{water}}{\mu_{water}} \times 1000$ [S10, unofficial]. Water has a HU value of 0, air -1000, and bone 500–1500 [S10, unofficial]. Implemented as a LUT where a color value is read out for a given HU value [S10, unofficial].

### Magnitude Color Gradient
A linear blue-green-red gradient stored in a 1D CUDA texture to visualize vector magnitude (velocity), where blue is lowest and red is highest [S11, unofficial]. It solves the problem of visualizing scalar fields derived from vector data [S11, unofficial]. The gradient is stored in a 1D texture on the GPU and queried using `tex1D` for interpolated colors [S11, unofficial].

### 3D Color Cube Interpolation
An interpolation method used in Smokeview version 4 and earlier where colors are interpolated directly within the 3D RGB color space [S2, unofficial]. It calculates the interpolated color as the average of two colors: $interpolated\ color = (f(c_1) + f(c_2))/2$ [S2, unofficial]. This is distinct from 1D TF sampling because it interpolates RGB values directly rather than indices [S2, unofficial]. Pitfall: Interpolating between red (1,0,0) and blue (0,0,1) yields purple (0.5,0.0,0.5), which is not in the colorbar and has no physical meaning [S2, unofficial].

### Gradient/Color Ramp Sampling
A method used in engines like Godot to read luminance in float and truncate each channel for the colour ramp shared with NoiseTexture2D [S12, unofficial]. It runs Gradient sampler and channelToByte in float [S12, unofficial]. Pitfall: A GradientTexture2D with `use_hdr` keeps a channel above 1 [S12, unofficial]. Measured ramp bytes off by one drop from 1212 to 0 of 2244 in Godot tests [S12, unofficial].

### Bilinear Image Sampling
Samples colors from a batch of images at specified 2D pixel coordinates using `torch.nn.functional.grid_sample` with `align_corners=True` and `mode="bilinear"` [S13, unofficial]. It takes an images tensor of shape `[batch_size, H, W, C]` and `xy_grid` of shape `[N, 2]` [S13, unofficial]. `align_corners=True` ensures corner pixels align with grid boundaries, preventing coordinate scaling artifacts at image edges [S13, unofficial].

### Labelmap Transfer Function LUT Construction
Constructs two 1D LUT textures (soft and hard) for labelmap transfer functions [S3, unofficial]. The soft LUT uses float precision and bilinear filtering; the hard LUT uses byte precision and point filtering [S3, unofficial]. It initializes float and byte pixel arrays, maps label indices to colors/alphas, converts float [0,1] to byte [0,255], and uploads them as `Texture2D` with appropriate wrap/filter modes [S3, unofficial]. `lutSize` is constant 256 [S3, unofficial].

### 2D Transfer Function
A volume rendering technique that maps two separate scalar fields to color and opacity using a 2D lookup table (image), extending the 1D transfer function approach [S14]. It solves the problem of classifying data based on multiple properties (e.g., value and gradient) [S14]. Unlike separable 1D transfer functions, the 2D TFN allows for non-separable, arbitrary 2D mappings [S14]. Pitfall: Ambiguity in mapping scalar fields to the 2D lookup table axes; a prescriptive interpretation is that the X-axis represents the scalar `value` and the Y-axis represents the gradient of the value, `grad(value)` [S14].

## Operating characteristics
*   **Time Complexity:** LUT construction is $O(N \cdot K)$ where $N$ is the LUT resolution and $K$ is the number of control points (or $O(N \log K)$ if using binary search for interval finding) [S7, unofficial]. Sampling is $O(1)$ per voxel via texture lookup [S1, book].
*   **Memory Complexity:** $O(N)$ for the LUT, where $N$ is typically 256–8192 entries [S6, S7, unofficial].
*   **Performance:** Hardware-accelerated bilinear filtering on 1D textures provides efficient, smooth interpolation [S4, S5, unofficial].
*   **Strengths:** Simple to implement, efficient, ensures colors stay within the defined colorbar (unlike 3D cube interpolation) [S2, unofficial].
*   **Limitations:** Only handles scalar data; cannot capture correlations between multiple data dimensions (requires 2D TF) [S1, book][S14]. Linear interpolation may not capture complex non-linear mappings [S4, unofficial].
*   **When to use:** Standard scalar volume visualization (CT, MRI, temperature fields) [S1, book][S10, unofficial].
*   **When not to use:** When data classification requires multiple scalar inputs (use 2D TF) or when high-order smoothness is required (use parametric generation) [S14, S5349].

## Background
The 1D transfer function is the simplest and most widely used transfer function in volume rendering, mapping the range of data values to color and opacity [S1, book][S9]. It is typically implemented with 1D texture lookup tables [S1, book][S9]. The concept of using 1D textures for color mapping to avoid artifacts associated with 3D color cube interpolation was highlighted in the development of tools like Smokeview, which switched from interpolating colors directly to interpolating indices within a 1D colorbar [S2, unofficial]. The opacity-weighted color model, where color components are multiplied by opacity, was established in early GPU rendering techniques (Wittenbrink et al. 1998) [S1, book][S9].

## Sources
[S1] Chapter 39. Volume Rendering Techniques | NVIDIA Developer, https://developer.nvidia.com/gpugems/gpugems/part-vi-beyond-triangles/chapter-39-volume-rendering-techniques, fetched 2026-10-26 (book)
[S2] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S3] guillaume-schneider/vrdf-sdk, https://github.com/guillaume-schneider/vrdf-sdk, fetched 2026-10-26 (unofficial)
[S4] faviasono/raycastingrendering, https://github.com/faviasono/raycastingrendering, fetched 2026-10-26 (unofficial)
[S5] Compressed volume rendering using deep learning, https://www.cs.umd.edu/sites/default/files/scholarly_papers/Jain.pdf, fetched 2026-10-26 (unofficial)
[S6] fraserlove/ossium, https://github.com/fraserlove/ossium, fetched 2026-10-27 (unofficial)
[S7] wilsonCernWq/TransferFunctionModule, https://github.com/wilsonCernWq/TransferFunctionModule, fetched 2026-10-27 (unofficial)
[S8] hopexn/VolumeRenderingParallel, https://github.com/hopexn/VolumeRenderingParallel, fetched 2026-10-27 (unofficial)
[S9] GPU Gems ch39, Volume rendering techniques, https://developer.nvidia.com/sites/all/modules/custom/gpugems/books/GPUGems/gpugems_ch39.html, fetched 2026-10-29
[S10] Volume Raycasting mit OpenCL, https://arxiv.org/abs/1609.01317v1, fetched 2026-10-26 (unofficial)
[S11] Realtime visualization of 3D vector fields using CUDA | Marek's blog, https://marekfiser.com/projects/realtime-visualization-of-3D-vector-fields-using-CUDA/, fetched 2026-10-28 (unofficial)
[S12] feat(renderer): draw FogVolume and FogMaterial into the volumetric fog, https://github.com/Vortiago/textscene-inspector/pull/732, fetched 2026-10-26 (unofficial)
[S13] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S14] KHR_VOLUME_TRANSFER_FUNCTION2D Extension, https://github.com/KhronosGroup/ANARI-Docs/pull/156, fetched 2026-10-27
