# 1D Transfer Function Generation and Sampling

Also called: TF1D, 1D Transfer Function

## Overview
1D Transfer Function Generation and Sampling maps scalar volume data values to RGBA color and opacity properties using a one-dimensional lookup table (LUT) [S1, book][S11]. It solves the problem of quantitative data visualization by assigning visual appearance to numerical voxel intensities [S4, unofficial]. Unlike [Parametric Function Generation](5349-parametric-function-generation.md) which computes values analytically, or [Color Blending](5150-color-blending.md) which composites final pixels, this method performs a deterministic, per-voxel assignment based on a 1D LUT [S1, book]. It builds on [Lookup Table Implementation](5347-lookup-table-implementation.md) by utilizing the generated LUT to index colors [S1, book].

## How it works
**Inputs and Outputs**
*   **Input:** A set of control points defining the transfer function. Each control point consists of a scalar position (domain value) and associated color (RGBA) and/or opacity values [S4, S8, unofficial]. Optional Gaussian objects may be added to modulate opacity [S8, unofficial].
*   **Output:** A 1D texture (LUT) of size $N$ (typically 256 or 4096 entries) containing pre-interpolated RGBA values [S1, book][S3, S8, unofficial].

**Data Structures**
*   **ControlPoint:** Stores a scalar `value` (float/int) and a `color` (TFColor) [S4, unofficial].
    *   `TFColor`: Fields `r`, `g`, `b`, `a` (all 32-bit floats, normalized to [0.0, 1.0]) [S4, S10, unofficial].
*   **AlphaControl:** Stores `pos` (vec2f) where `pos.x` is the scalar value and `pos.y` is the opacity [S8, unofficial].
*   **GaussianObject:** Stores `mean` (float), `sigma` (float), `heightFactor` (float), and an `alphaArray` (vector<float> of size $N$) [S8, unofficial].
*   **LUT (m_rgbaTable):** A contiguous array of `vec4f` (R, G, B, A) of size $N$ [S8, S10, unofficial].

**Algorithm Steps**
1.  **Initialization:** Create an empty list of color control points and alpha control points. If no control points are provided, insert a default point at position 0.0 with color (0,0,0,0) [S8, unofficial].
2.  **Sorting:** Sort the color control points by their scalar position in ascending order [S8, unofficial].
3.  **LUT Construction:** Iterate through each index $i$ from $0$ to $N-1$ of the LUT:
    a.  **Domain Mapping:** Calculate the normalized scalar value $v$ corresponding to index $i$.
        *   *Center-aligned (Color):* $v = \frac{i + 0.5}{N}$ [S8, unofficial].
        *   *Edge-aligned (Alpha):* $v = \frac{i}{N - 1}$ [S8, unofficial].
    b.  **Interval Search:** Find the two adjacent control points, `left` and `right`, such that `left.position` $\le v \le$ `right.position` [S8, unofficial].
    c.  **Boundary Clamping:** If $v$ is less than the first control point's position, use the first control point. If $v$ is greater than the last, use the last [S8, unofficial].
    d.  **Interpolation Weight:** Calculate the interpolation weight $w$:
        $$ w = \frac{|v - p_L|}{|p_R - p_L|} $$
        where $p_L$ and $p_R$ are the positions of the left and right control points [S8, unofficial].
    e.  **Color Interpolation:** Interpolate the RGB components:
        $$ c = \text{mix}(c_L, c_R, w) $$
        where $c_L$ and $c_R$ are the colors of the left and right control points [S8, unofficial].
    f.  **Alpha Interpolation:** Interpolate the alpha value $\alpha$ using the same weight $w$ and alpha control points [S8, unofficial].
    g.  **Gaussian Modulation:** If Gaussian objects are present, update the alpha channel of the LUT entry:
        $$ \text{m\_rgbaTable}[i].w = \max(\text{m\_rgbaTable}[i].w, \text{gaussianObjects}[j].\text{alphaArray}[i]) $$
        This uses maximum compositing, allowing Gaussian opacity to override but not attenuate control-point opacity [S8, unofficial].
    h.  **Storage:** Store the resulting RGBA vector in `m_rgbaTable[i]` [S8, unofficial].
4.  **Texture Upload:** Upload the `m_rgbaTable` to GPU memory as a 1D texture (e.g., `GL_TEXTURE_1D` or `GL_TEXTURE_2D` with height 1) with format `RGBAFloat` (or `RGBA8` for byte precision) and linear filtering [S3, S8, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_1d_lut(control_points, alpha_controls, gaussian_objects, resolution=256):
    """
    Generates a 1D LUT from control points.
    control_points: List of (position, r, g, b, a) tuples, sorted by position.
    alpha_controls: List of (position, alpha) tuples, sorted by position.
    gaussian_objects: List of GaussianObject instances.
    resolution: Size of the LUT (N).
    """
    if not control_points:
        control_points = [(0.0, 0.0, 0.0, 0.0, 0.0)]
    
    # Ensure sorted
    control_points.sort(key=lambda p: p[0])
    alpha_controls.sort(key=lambda p: p[0])
    
    lut = [None] * resolution
    
    for i in range(resolution):
        # 1. Map index to normalized domain value [0, 1]
        # Using center-aligned binning for color consistency [S8, unofficial]
        v = (i + 0.5) / resolution
        
        # 2. Find interval for Color
        left_cp, right_cp = find_interval(control_points, v)
        
        # 3. Interpolate Color
        if left_cp is None:
            color = control_points[0]
        elif right_cp is None:
            color = control_points[-1]
        else:
            p_l, p_r = left_cp[0], right_cp[0]
            if p_r == p_l:
                w = 0.0
            else:
                w = abs(v - p_l) / abs(p_r - p_l)
            
            r = lerp(left_cp[1], right_cp[1], w)
            g = lerp(left_cp[2], right_cp[2], w)
            b = lerp(left_cp[3], right_cp[3], w)
            a = lerp(left_cp[4], right_cp[4], w)
            color = (r, g, b, a)
            
        # 4. Find interval for Alpha (using edge-aligned binning [S8, unofficial])
        v_alpha = i / (resolution - 1) if resolution > 1 else 0.0
        left_ac, right_ac = find_interval(alpha_controls, v_alpha)
        
        if left_ac is None:
            alpha = alpha_controls[0][1]
        elif right_ac is None:
            alpha = alpha_controls[-1][1]
        else:
            p_l, p_r = left_ac[0], right_ac[0]
            if p_r == p_l:
                w = 0.0
            else:
                w = abs(v_alpha - p_l) / abs(p_r - p_l)
            alpha = lerp(left_ac[1], right_ac[1], w)
            
        # 5. Apply Gaussian Modulation (Max compositing)
        final_alpha = max(color[3], alpha)
        for gauss in gaussian_objects:
            # Gaussian alpha array is pre-computed for the resolution
            gauss_alpha = gauss.alpha_array[i]
            final_alpha = max(final_alpha, gauss_alpha)
            
        lut[i] = (color[0], color[1], color[2], final_alpha)
        
    return lut

def find_interval(points, value):
    """Finds the two adjacent points surrounding 'value'."""
    if not points:
        return None, None
    if value <= points[0][0]:
        return points[0], points[0]
    if value >= points[-1][0]:
        return points[-1], points[-1]
    
    for i in range(len(points) - 1):
        p1 = points[i]
        p2 = points[i+1]
        if p1[0] <= value <= p2[0]:
            return p1, p2
    return points[-2], points[-1]

def lerp(a, b, t):
    return a + t * (b - a)
```

**Parameters**
*   `resolution`: Size of the LUT. Default 1024 [S8, unofficial], common values 256 for 8-bit data [S1, book] or 4096 for higher precision [S3, unofficial].
*   `brightness_factor`: Multiplier for RGB channels (optional) [S10, unofficial].
*   `tf_window_left` / `tf_window_width`: Define the domain range $[x_{min}, x_{max}]$ for normalization [S2, unofficial].

**Edge Cases and Pitfalls**
*   **Division by Zero:** If adjacent control points have identical positions, the interpolation weight calculation divides by zero. Guard against this by checking `p_r == p_l` [S7, S8, unofficial].
*   **Alpha Normalization:** Alpha values should be rounded to two decimal places before storage to reduce precision artifacts: `alpha = floor(alpha * 100) / 100.0` [S4, unofficial].
*   **Binning Misalignment:** `updateColorMap` uses center-aligned bins `(i+0.5)/resolution`, while `updateFromAlphaControls` uses edge-aligned values `i/(resolution-1)`. This causes a slight misalignment between color and alpha sampling at boundaries [S8, unofficial].
*   **Opacity-Weighted Color:** Color components must be multiplied by opacity during rendering because color approximates both emission and absorption [S1, book][S11].

## Subtopics

### Hounsfield Scale Color Mapping
A method to assign color values to voxels based on attenuation coefficients (e.g., from CT scans) using the Hounsfield scale [S12, unofficial]. It is implemented as a LUT where a color value is read out for a given HU value [S12, unofficial]. The Hounsfield unit is calculated as $HU = \frac{\mu_{tissue} - \mu_{water}}{\mu_{water}} \times 1000$ [S12, unofficial]. Water has a HU value of 0, and bone ranges from 500 to 1500 [S12, unofficial].

### 3D Color Cube Interpolation
An interpolation method used in Smokeview version 4 and earlier where colors are interpolated directly within the 3D RGB color space [S2, unofficial]. The interpolated color is calculated as the average of two colors: $interpolated\ color = (f(c_1) + f(c_2))/2$ [S2, unofficial]. This causes artifacts near large gradients (e.g., fire) because interpolating between red (1,0,0) and blue (0,0,1) yields purple (0.5,0.0,0.5), which is not in the colorbar and has no physical meaning [S2, unofficial].

### Gradient/Color Ramp Sampling
Reads luminance in float and truncates each channel for the colour ramp shared with NoiseTexture2D, as Godot does [S13, unofficial]. Runs Gradient sampler and channelToByte in float [S13, unofficial]. A pitfall is that a GradientTexture2D with `use_hdr` keeps a channel above 1 [S13, unofficial]. Godot-measured ramp bytes off by one drop from 1212 to 0 of 2244 [S13, unofficial].

### Bilinear Image Sampling
Samples colors from a batch of images at specified 2D pixel coordinates [S14, unofficial]. Takes `images` tensor of shape `[batch_size, H, W, C]` and `xy_grid` of shape `[N, 2]` [S14, unofficial]. Flips `xy_grid` sign, reshapes to `[batch_size, N, 1, 2]` [S14, unofficial]. Permutes images to `[batch_size, C, H, W]` [S14, unofficial]. Uses `torch.nn.functional.grid_sample` with `align_corners=True` and `mode="bilinear"` [S14, unofficial]. Returns sampled colors reshaped to `[N, C]` [S14, unofficial]. `align_corners=True` ensures corner pixels align with grid boundaries during interpolation, preventing coordinate scaling artifacts at image edges [S14, unofficial].

### Labelmap Transfer Function LUT Construction
Constructs two 1D LUT textures (soft and hard) for labelmap transfer functions [S3, unofficial]. The soft LUT uses float precision and bilinear filtering; the hard LUT uses byte precision and point filtering [S3, unofficial]. Initializes `floatPixels` and `bytePixels` arrays of size 256 [S3, unofficial]. Iterates over entries, calculates index `idx = clamp(round(label), 0, 255)`, extracts RGBA, and stores in `floatPixels` [S3, unofficial]. Converts float RGBA [0,1] to byte [0,255] by multiplying by 255, rounding, and clamping [S3, unofficial]. Creates `lutSoft` (Texture2D, RGBAFloat, Bilinear) and `lutHard` (Texture2D, RGBA32, Point) [S3, unofficial].

### 2D Transfer Function
A volume rendering technique that maps two separate scalar fields to color and opacity using a 2D lookup table (image), extending the 1D transfer function approach [S15]. The X-axis represents the scalar `value` and the Y-axis represents the gradient of the value, `grad(value)` [S15]. Unlike separable 1D transfer functions which combine two 1D lookups (potentially saving memory), the 2D TFN allows for non-separable, arbitrary 2D mappings without implementation constraints on combination [S15].

### Material Synchronization LUT Adjustment
An algorithm to adjust a 1D Look-Up Table (LUT) mapping sample values to material indices, ensuring that the distribution of sample values across materials is synchronized with the frame number and material centers [S16, unofficial]. Calculates the common base allocation range `d` as `frameNumber / colorKind` [S16, unofficial]. Initializes material center positions `indexM[i]` to `d*i + d / 2` for each material `i` from 0 to `colorKind - 1` [S16, unofficial].

### BuildFunctionFromTable
Constructs a color transfer function from a pre-existing table of colors [S17]. The function range will be `[x1, x2]` and the function will have size `nodes` [S17]. Function values will be regularly spaced between `x1` and `x2` [S17]. The table parameter's layout is assumed to be `[R1, G1, B1, R2, G2, B2, ..., Rn, Gn, Bn]`, and it is assumed to be a block of memory of size `[3*size]` [S17].

### Scalar Mapping Output Formats
The output format for mapped scalars can be set to specific component counts [S17]. Default output format is RGBA if not supplied [S17].

## Operating characteristics
*   **Time Complexity:** LUT generation is $O(N \cdot C)$ where $N$ is the LUT resolution and $C$ is the number of control points (due to interval search). Sampling is $O(1)$ per voxel using hardware interpolation [S1, book][S5, unofficial].
*   **Memory Complexity:** $O(N)$ for the LUT storage [S8, unofficial].
*   **Performance:** Hardware interpolation (linear filtering) is efficient on GPUs [S5, unofficial]. Direct lookup requires integer index calculation and clamping [S3, S5, unofficial].
*   **Strengths:** Simple to implement, fast runtime sampling, supports smooth color transitions via interpolation [S4, unofficial].
*   **Limitations:** Only maps a single scalar value; cannot capture correlations between multiple properties (e.g., value and gradient) without 2D TFs [S1, book]. Linear interpolation may produce artifacts in color space if not careful (see 3D Color Cube Interpolation pitfall) [S2, unofficial].
*   **When to use:** When visualization depends primarily on scalar intensity (e.g., CT density, temperature) [S1, book].
*   **When not to use:** When classification requires multi-dimensional data (e.g., diffusion tensor imaging) [S1, book].

## Background
The 1D transfer function was introduced as a fundamental technique in volume rendering to map scalar data to optical properties [S1, book]. It is widely used in medical imaging (CT, MRI) and scientific visualization [S1, book][S12, unofficial]. The method evolved from simple color mapping to include interactive editors with control points and Gaussian opacity modifiers [S4, S8, unofficial]. The distinction between interpolating color indices (1D texture) versus interpolating RGB values directly (3D cube) was highlighted in Smokeview documentation to avoid color artifacts [S2, unofficial].

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
