# Scaling and Biasing

Also called: BuildScaleBiasProgram, Leveling, Window/Level (Windowing) Adjustment, Windowing and Leveling

## Overview
Scaling and Biasing is a volume normalization technique that applies a linear transformation to scalar field values to optimize the display of specific tissues or structures within volumetric images [S4, unofficial]. It solves the problem of limited display dynamic range by mapping a user-defined intensity window to the full output range [S1, unofficial]. Unlike [Power Transformation](2346-power-transformation.md), which applies non-linear gamma curves, Scaling and Biasing preserves linear relationships within the selected window, making it the standard method for medical imaging windowing [S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar value $x$ from the volume texture sample.
*   **Parameters:**
    *   $W$ (Window Width): The range of intensities to map to the display [S2, unofficial]. Default: 1048 [S2, unofficial].
    *   $L$ (Window Level/Center): The center of the intensity window [S2, unofficial]. Default: 0 [S2, unofficial].
*   **Output:** A normalized scalar value $y$ suitable for opacity/color mapping [S3, unofficial].

**Data Structures:**
The method is typically implemented as a fragment shader program or a uniform update in a rendering engine.
*   **Shader Uniforms:** `windowWidth` (float), `windowLevel` (float) [S2, unofficial].
*   **Engine State:** In some implementations, the volume object holds these parameters directly, e.g., `cptr->volume` or `volume.setWindowLevel()` [S2, S3, unofficial].

**Algorithm:**
The transformation is a linear scaling followed by a bias (offset). The standard window/level formula maps the interval $[L - W/2, L + W/2]$ to $[0, 1]$ (or $[0, 255]$ for 8-bit displays).

1.  Calculate the lower bound of the window: $min = L - W/2$.
2.  Calculate the upper bound of the window: $max = L + W/2$.
3.  Compute the scaled value:
    $$ y = \frac{x - min}{max - min} $$
    Substituting $min$ and $max$:
    $$ y = \frac{x - (L - W/2)}{W} $$
    $$ y = \frac{x - L}{W} + 0.5 $$
4.  Clamp the result to the valid output range (typically $[0, 1]$ or $[0, 255]$).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_scaling_and_biasing(x: float, window_width: float, window_level: float) -> float:
    """
    Applies window/level scaling and biasing to a scalar value.
    
    Args:
        x: Input scalar value from volume texture.
        window_width: The range of intensities to map (W).
        window_level: The center of the intensity window (L).
        
    Returns:
        Normalized value in [0.0, 1.0].
    """
    if window_width <= 0:
        return 0.0 # Prevent division by zero
        
    # Calculate the center offset relative to the window center
    # Formula: y = (x - L) / W + 0.5
    normalized = (x - window_level) / window_width + 0.5
    
    # Clamp to [0.0, 1.0]
    if normalized < 0.0:
        return 0.0
    elif normalized > 1.0:
        return 1.0
    else:
        return normalized
```

**Edge Cases and Pitfalls:**
*   **Zero Width:** If $W=0$, the division is undefined. The implementation must handle this, typically by returning 0 or the previous value [S2, unofficial].
*   **Precision:** In fixed-point or integer arithmetic (e.g., embedded systems or specific voxel engines), intermediate calculations may overflow. For example, in the Comanche engine, volume adjustments are limited to specific bit masks (e.g., `effect & 0xF`) and clamped to legal volumes (0..64) [S3, unofficial].
*   **Boundary Conditions:** Values outside the window are clamped. This creates a "flat" appearance for structures outside the window of interest, which is intentional for contrast optimization [S4, unofficial].

## Subtopics

### Volume Slide Down
A variant found in specific voxel terrain engines (e.g., Comanche) where "volume" refers to a discrete sample intensity or height value rather than a continuous medical scalar.
*   **Definition:** Adjusts the current sample's volume by incrementing or decrementing it by a fixed amount $x$ at the beginning of a division operation [S3, unofficial].
*   **Mechanism:**
    *   **Increment:** `cptr->volume += (effect & 0xF)`. Clamped to a maximum of 64 [S3, unofficial].
    *   **Decrement:** `cptr->volume -= (effect & 0xF)`. If volume exceeds 200 (likely a typo in source or specific state check), it resets to 0 [S3, unofficial].
*   **Pitfalls:** Cannot slide beyond volume 64 (for increment) or 0 (for decrement) [S3, unofficial].
*   **When to choose:** Only for discrete voxel terrain height/layer manipulation in legacy or specific game engines, not for general medical volume rendering.

### Texture Format Remapping
A preprocessing step for GPU texture storage formats.
*   **Definition:** Converts uint16 data stored in `r16float` textures back to normalized float values for rendering [S5, unofficial].
*   **Mechanism:** If the GPU texture format is `r16float` and the metadata indicates the data is not native float, the renderer applies a scale and bias:
    *   `texelScale = 65535`
    *   `texelBias = -UINT16_FLOAT16_OFFSET`
    *   The renderer maps texels to `value / 65535` [S5, unofficial].
*   **Parameters:** `UINT16_FLOAT16_OFFSET` is a constant offset; exact value not given in sources [S5, unofficial].
*   **When to choose:** When storing 16-bit integer volume data in 16-bit floating-point textures to save memory/bandwidth, requiring a reverse transformation during sampling.

### Level Correction and Gamma Correction Filter
A non-linear variant that corrects data values according to a gamma curve.
*   **Definition:** Corrects data values according to a gamma curve. Works on the data itself, not the rendered image [S6].
*   **Mechanism:** Uses IN, OUT, and Gamma parameters. The last value on the right of the gamma curve is the offset value [S6]. The correction curve is shown in the preview preference [S6].
*   **Comparison:** Unlike filters that only affect preview, this filter works in both preview and PovRay rendering [S6].
*   **When to choose:** When non-linear contrast adjustment is required on the source data before rendering, particularly for PovRay compatibility.

### Range Remapping
A shader-level remapping used in production renderers like Redshift.
*   **Definition:** Adjusts OpenVDB channel values (density/emission) by remapping an old min/max range to a new min/max range before use [S7].
*   **Mechanism:**
    1.  Identify old min and old max (e.g., 0 and 100 for temperature) [S7].
    2.  Set new min and new max (e.g., 0 and 1 for emission color ramp) [S7].
    3.  Remap values: $y = \frac{x - old\_min}{old\_max - old\_min} \times (new\_max - new\_min) + new\_min$ [S7].
*   **When to choose:** When volume data ranges (e.g., 0-100) do not match the expected input range of a shader node or color ramp (e.g., 0-1).

## Operating characteristics
*   **Time Complexity:** $O(1)$ per voxel/sample. The operation is a simple arithmetic expression.
*   **Memory Complexity:** $O(1)$ additional memory for parameters ($W, L$).
*   **Performance:** Extremely fast; often executed in the fragment shader or vertex shader with negligible overhead [S1, unofficial].
*   **Strengths:**
    *   Provides immediate visual feedback for contrast adjustment [S4, unofficial].
    *   Standard in medical imaging for isolating specific tissue densities [S4, unofficial].
    *   Linear preservation within the window allows for accurate intensity interpretation within the ROI [S4, unofficial].
*   **Limitations:**
    *   Does not enhance features outside the window; they are clipped to black or white [S4, unofficial].
    *   Linear transformation may not be optimal for data with non-Gaussian noise or complex histograms compared to [Power Transformation](2346-power-transformation.md) [S4, unofficial].
*   **When to use:** When the user needs to inspect specific intensity ranges (e.g., bone vs. soft tissue) in medical or scientific data.
*   **When not to use:** When global contrast enhancement is needed across the entire histogram, or when non-linear contrast is desired.

## Background
Scaling and Biasing, commonly known as Window/Level or Windowing, is a fundamental technique in medical imaging visualization [S4, unofficial]. It allows radiologists to "look inside" objects by adjusting contrast and brightness to optimize the display of specific tissues [S4, unofficial]. The technique is implemented in various VTK-based renderers, such as `VTKMultiVolumeRayCaster`, which generates fragment shader programs for this purpose [S1, unofficial]. It is also a standard UI feature in volume rendering applications, where sliders adjust `windowWidth` and `windowLevel` parameters, triggering viewport invalidation [S2, unofficial].

## Sources
[S1] bozorgi/VTKMultiVolumeRayCaster, https://github.com/bozorgi/VTKMultiVolumeRayCaster, fetched 2026-10-26 (unofficial)
[S2] termijn/webgl-volumerendering, https://github.com/termijn/webgl-volumerendering, fetched 2026-10-27 (unofficial)
[S3] AndreasVlcek/Voxel-Space-Engine-Comanche-Terrain-Rendering, https://github.com/AndreasVlcek/Voxel-Space-Engine-Comanche-Terrain-Rendering, fetched 2026-10-27 (unofficial)
[S4] ChatGPT version of Ct Volume Rendering, https://www.flyriver.com/s/ct-volume-rendering, fetched 2026-10-29 (unofficial)
[S5] Show HN: Kiln – WebGPU-native out-of-core volume rendering for multi-GB datasets, https://github.com/MPanknin/kiln-render, fetched 2026-10-26 (unofficial)
[S6] DF3 Viewer Oosawa, https://qcganime.web.fc2.com/OOSAWA/MANUAL/FilterE.html, fetched 2026-10-29
[S7] Volume Rendering - Maxon, https://help.maxon.net/c4d/s26/en-us/Content/_REDSHIFT_/html/Volume%2BRendering.html, fetched 2026-10-26
