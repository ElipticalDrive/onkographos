# 1D Transfer Function Generation and Sampling

Also called: TF1D, 1D Transfer Function

## Overview
1D Transfer Function Generation and Sampling maps scalar volume data values to RGBA color and opacity properties using a one-dimensional lookup table (LUT) [S1, book][S11]. It solves the problem of quantitative data visualization by assigning visual appearance to numerical voxel intensities [S4, unofficial]. Unlike [Parametric Function Generation](5349-parametric-function-generation.md) which computes values analytically, or [Color Blending](5150-color-blending.md) which composites final pixels, this method performs a deterministic, per-voxel assignment based on a 1D LUT [S1, book]. It builds on [Lookup Table Implementation](5347-lookup-table-implementation.md) by utilizing the generated LUT to index colors [S1, book].

## How it works
**Inputs and Outputs**
*   **Input:** A set of control points defining the transfer function. Each control point consists of a scalar position (value) and associated RGBA color and opacity values [S4, S8, unofficial]. Optional Gaussian objects may be added to modulate opacity [S8, unofficial].
*   **Output:** A 1D texture (LUT) of size $N$ (commonly 256 or 4096 entries) containing interpolated RGBA values [S1, book][S3, S8, unofficial].

**Data Structures**
*   **ControlPoint:** Stores `value` (scalar/int) and `color` (TFColor) [S4, unofficial].
*   **TFColor:** Stores `r`, `g`, `b`, `a` (all floats in [0.0, 1.0]) [S4, unofficial].
*   **LUT Array:** A contiguous array of `float4` (R, G, B, A) of size $N$ [S2, unofficial][S5, unofficial].
*   **GaussianObject (Optional):** Stores `mean`, `sigma`, `heightFactor`, and an `alphaArray` of size $N$ [S8, unofficial].

**Steps**
1.  **Initialization:** Create an empty list of control points. If no control points are provided, insert a default point at position 0.0 with color (0,0,0,0) [S8, unofficial].
2.  **Sorting:** Sort the control points by their scalar position in ascending order [S4, S8, unofficial].
3.  **LUT Construction:** Iterate through each index $i$ from $0$ to $N-1$ of the LUT:
    a.  Compute the normalized scalar value $v$ corresponding to index $i$. Two common binning strategies exist:
        *   Center-aligned: $v = \frac{i + 0.5}{N}$ [S8, unofficial].
        *   Edge-aligned: $v = \frac{i}{N-1}$ [S8, unofficial].
        *   *Note:* Inconsistent binning between color and alpha updates can cause boundary misalignment [S8, unofficial].
    b.  Find the interval $[p_L, p_R]$ in the sorted control points such that $p_L \le v < p_R$ [S8, unofficial].
    c.  **Boundary Clamping:** If $v \le p_L$ (or no left neighbor), use the color/alpha of the first control point. If $v \ge p_R$ (or no right neighbor), use the color/alpha of the last control point [S8, unofficial].
    d.  **Interpolation Weight:** Calculate the interpolation weight $w$:
        $$ w = \frac{|v - p_L|}{|p_R - p_L|} $$
        where $p_L$ and $p_R$ are the positions of the left and right control points [S8, unofficial].
    e.  **Color Interpolation:** Interpolate RGB components:
        $$ c = \text{mix}(c_L, c_R, w) $$
        where $c_L$ and $c_R$ are the colors of the left and right control points [S8, unofficial].
    f.  **Alpha Interpolation:** Interpolate alpha values similarly:
        $$ \alpha = \text{mix}(\alpha_L, \alpha_R, w) $$
        [S8, unofficial].
    g.  **Gaussian Modulation:** If Gaussian objects are present, update the alpha channel using the maximum of the interpolated alpha and the Gaussian alpha at index $i$:
        $$ \alpha_{final} = \max(\alpha, \text{GaussianAlpha}[i]) $$
        [S8, unofficial].
    h.  Store the resulting RGBA vector in the LUT at index $i$ [S8, unofficial].
4.  **Normalization:** Ensure RGB components are in [0.0, 1.0] (divide by 255.0 if input is 0-255) [S4, unofficial]. Alpha values may be rounded to two decimal places for storage precision [S4, unofficial].
5.  **Upload:** Upload the LUT array to GPU memory as a 1D texture with linear filtering and clamp addressing [S5, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_1d_lut(control_points: List[ControlPoint], resolution: int = 256) -> List[float4]:
    """
    Generates a 1D LUT from control points.
    ControlPoint: {value: float, color: {r: float, g: float, b: float, a: float}}
    Returns: List of float4 [r, g, b, a]
    """
    if not control_points:
        control_points = [ControlPoint(value=0.0, color={'r':0, 'g':0, 'b':0, 'a':0})]
    
    # Sort by value
    control_points.sort(key=lambda cp: cp.value)
    
    n = len(control_points)
    lut = [None] * resolution
    
    for i in range(resolution):
        # Center-aligned binning [S8, unofficial]
        v = (i + 0.5) / resolution
        
        # Find interval
        # upper_bound returns first element > v
        upper_idx = 0
        for idx, cp in enumerate(control_points):
            if cp.value > v:
                upper_idx = idx
                break
        else:
            upper_idx = n # v is greater than all points
        
        if upper_idx == 0:
            # Clamp to first point
            lut[i] = float4(control_points[0].color['r'], control_points[0].color['g'], 
                            control_points[0].color['b'], control_points[0].color['a'])
        elif upper_idx >= n:
            # Clamp to last point
            lut[i] = float4(control_points[n-1].color['r'], control_points[n-1].color['g'], 
                            control_points[n-1].color['b'], control_points[n-1].color['a'])
        else:
            # Interpolate between upper_idx-1 and upper_idx
            left = control_points[upper_idx - 1]
            right = control_points[upper_idx]
            
            denom = right.value - left.value
            if denom == 0:
                # Guard against division by zero [S7, unofficial]
                w = 0.0
            else:
                w = abs(v - left.value) / denom
            
            r = left.color['r'] + w * (right.color['r'] - left.color['r'])
            g = left.color['g'] + w * (right.color['g'] - left.color['g'])
            b = left.color['b'] + w * (right.color['b'] - left.color['b'])
            a = left.color['a'] + w * (right.color['a'] - left.color['a'])
            
            lut[i] = float4(r, g, b, a)
            
    return lut

def sample_lut(lut: List[float4], value: float, v_min: float, v_max: float) -> float4:
    """
    Samples the LUT for a given scalar value.
    Uses hardware interpolation logic (linear) or direct lookup.
    """
    # Normalize value to [0, 1]
    if v_max == v_min:
        x = 0.0
    else:
        x = (value - v_min) / (v_max - v_min)
    
    # Clamp to [0, 1]
    x = max(0.0, min(1.0, x))
    
    # Direct lookup with linear interpolation (simulating hardware bilinear/linear filter)
    # Index in LUT
    idx_float = x * (len(lut) - 1)
    idx0 = int(idx_float)
    idx1 = min(idx0 + 1, len(lut) - 1)
    t = idx_float - idx0
    
    c0 = lut[idx0]
    c1 = lut[idx1]
    
    r = c0.r + t * (c1.r - c0.r)
    g = c0.g + t * (c1.g - c0.g)
    b = c0.b + t * (c1.b - c0.b)
    a = c0.a + t * (c1.a - c0.a)
    
    return float4(r, g, b, a)
```

**Parameters**
*   `resolution`: Size of the LUT. Default 1024 [S8, unofficial], common values 256 for 8-bit data [S1, book] or 4096 for higher precision [S3, unofficial].
*   `brightness_factor`: Multiplier for RGB channels, default 1.0 [S10, unofficial].
*   `v_min`, `v_max`: Data range for normalization [S2, unofficial].

**Edge Cases and Pitfalls**
*   **Division by Zero:** If adjacent control points have identical scalar values, the interpolation weight denominator becomes zero [S7, unofficial].
*   **Color Artifacts:** Interpolating colors directly in RGB space (3D cube) instead of interpolating indices or using 1D textures can produce unintended colors (e.g., purple between red and blue) [S2, unofficial].
*   **Alpha Combination:** Using `max` for Gaussian alpha modulation overrides control opacity rather than blending it [S8, unofficial].
*   **Binning Misalignment:** Using center-aligned bins for color and edge-aligned bins for alpha causes slight misalignment at boundaries [S8, unofficial].

## Subtopics

### Hounsfield Scale Color Mapping
A method to assign color values to voxels based on attenuation coefficients (e.g., from CT scans) using the Hounsfield scale [S12, unofficial]. It implements a LUT where a color value is read out for a given HU value [S12, unofficial]. Water has a HU value of 0, and bone ranges from 500 to 1500 HU [S12, unofficial].

### 3D Color Cube Interpolation
An interpolation method used in Smokeview version 4 and earlier where colors are interpolated directly within the 3D RGB color space [S2, unofficial]. The interpolated color is calculated as the average of two colors: $interpolated color = (f(c_1) + f(c_2))/2$ [S2, unofficial]. This causes artifacts near large gradients (e.g., fire) because the interpolated color (e.g., purple between red and blue) is not in the defined colorbar and has no physical meaning [S2, unofficial].

### Gradient/Color Ramp Sampling
Reads luminance in float and truncates each channel for the colour ramp shared with NoiseTexture2D, as Godot does [S13, unofficial]. Runs Gradient sampler and channelToByte in float [S13, unofficial]. A pitfall is that a GradientTexture2D with use_hdr keeps a channel above 1 [S13, unofficial].

### Bilinear Image Sampling
Samples colors from a batch of images at specified 2D pixel coordinates using `torch.nn.functional.grid_sample` with `align_corners=True` and `mode="bilinear"` [S14, unofficial]. `align_corners=True` ensures corner pixels align with grid boundaries during interpolation, preventing coordinate scaling artifacts at image edges [S14, unofficial].

### Labelmap Transfer Function LUT Construction
Constructs two 1D LUT textures (soft and hard) for labelmap transfer functions [S3, unofficial]. The soft LUT uses float precision and bilinear filtering; the hard LUT uses byte precision and point filtering [S3, unofficial]. It initializes float and byte pixel arrays, maps label indices to colors/alphas, converts float [0,1] to byte [0,255], and uploads them as Texture2D with appropriate wrap/filter modes [S3, unofficial].

### 2D Transfer Function
A volume rendering technique that maps two separate scalar fields to color and opacity using a 2D lookup table (image), extending the 1D transfer function approach [S15]. The X-axis typically represents the scalar `value` and the Y-axis represents the gradient of the value, `grad(value)` [S15]. Unlike separable 1D transfer functions, the 2D TFN allows for non-separable, arbitrary 2D mappings [S15].

### Material Synchronization LUT Adjustment
An algorithm to adjust a 1D LUT mapping sample values to material indices, ensuring that the distribution of sample values across materials is synchronized with the frame number and material centers [S16, unofficial]. It calculates a common base allocation range and initializes material center positions [S16, unofficial].

### BuildFunctionFromTable
Constructs a color transfer function from a pre-existing table of colors [S17]. The function range becomes [x1, x2], the function size becomes `nodes`, and function values are regularly spaced between x1 and x2 [S17]. The table layout is assumed to be [R1, G1, B1, R2, G2, B2, ..., Rn, Gn, Bn] [S17].

### Scalar Mapping Output Formats
The output format for mapped scalars can be set to specific component counts, with RGBA being the default if not supplied [S17].

## Operating characteristics
*   **Time Complexity:** LUT construction is $O(N \cdot C)$ where $N$ is the LUT resolution and $C$ is the number of control points (due to sorting and interval search). Sampling is $O(1)$ per voxel.
*   **Memory Complexity:** $O(N)$ for the LUT storage.
*   **Performance:** Hardware interpolation (texture sampling) is extremely fast on GPUs [S5, unofficial]. Direct lookup is also efficient but lacks hardware filtering [S3, S5, S8, unofficial].
*   **Strengths:** Simple to implement, fast rendering, intuitive user interface (control points) [S5, S8, unofficial].
*   **Limitations:** Limited to scalar data only; cannot capture correlations between multiple scalar fields (requires 2D TF) [S1, book][S15]. Linear interpolation may not capture complex feature boundaries as well as higher-order methods [S4, unofficial].
*   **When to use:** Standard volume rendering of single-scalar datasets (CT, MRI) [S1, book][S11].
*   **When not to use:** When data requires multi-dimensional classification (e.g., value and gradient) [S1, book][S15].

## Background
The method originated as a standard technique in volume rendering to map data values to optical properties [S1, book]. It was widely adopted in early GPU-based volume rendering pipelines, such as those described in the NVIDIA GPU Gems [S1, book]. The distinction between interpolating color indices (1D texture) versus interpolating colors in 3D space was highlighted in tools like Smokeview to avoid visual artifacts [S2, unofficial]. The technique is implemented in various libraries and frameworks, including VTK [S17], Unity [S3, unofficial], and custom research code [S4, unofficial][S8, unofficial].

## Sources
[S1] Chapter 39. Volume Rendering Techniques | NVIDIA Developer, https://developer.nvidia.com/gpugems/gpugems/part-vi-beyond-triangles/chapter-39-volume-rendering-techniques, fetched 2026-10-26 (book)
[S2] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S3] guillaume-schneider/vrdf-sdk, https://github.com/guillaume-schneider/vrdf-sdk, fetched 2026-10-26 (unofficial)
[S4] faviasono/raycastingrendering, https://github.com/faviasono/raycastingrendering, fetched 2026-10-26 (unofficial)
[S5] Compressed volume rendering using deep learning, https://www.cs.umd.edu/sites/default/files/scholarly_papers/Jain.pdf, fetched 2026-10-26 (unofficial)
[S6] fraserlove/ossium, https://github.com/fraserlove/ossium, fetched 2026-10-27 (unofficial)
[S7] ragmar/VolumeRender, https://github.com/ragmar/VolumeRender, fetched 2026-10-27 (unofficial)
[S8] wilsonCernWq/TransferFunctionModule, https://github.com/wilsonCernWq/TransferFunctionModule, fetched 2026-10-27 (unofficial)
[S9] hopexn/VolumeRenderingParallel, https://github.com/hopexn/VolumeRenderingParallel, fetched 2026-10-27 (unofficial)
[S10] GitHub - priscilla100/medical_volume_renderer, https://github.com/priscilla100/medical_volume_renderer/, fetched 2026-10-29 (unofficial)
[S11] GPU Gems ch39, Volume rendering techniques, https://developer.nvidia.com/sites/all/modules/custom/gpugems/books/GPUGems/gpugems_ch39.html, fetched 2026-10-29
[S12] Volume Raycasting mit OpenCL, https://arxiv.org/abs/1609.01317v1, fetched 2026-10-26 (unofficial)
[S13] feat(renderer): draw FogVolume and FogMaterial into the volumetric fog, https://github.com/Vortiago/textscene-inspector/pull/732, fetched 2026-10-26 (unofficial)
[S14] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S15] KHR_VOLUME_TRANSFER_FUNCTION2D Extension, https://github.com/KhronosGroup/ANARI-Docs/pull/156, fetched 2026-10-27
[S16] RubbishKing0624/VolumeRendering, https://github.com/RubbishKing0624/VolumeRendering, fetched 2026-10-26 (unofficial)
[S17] vtkColorTransferFunctions, http://www.vtk.org/doc/nightly/html/classvtkColorTransferFunction.html, fetched 2026-10-29
