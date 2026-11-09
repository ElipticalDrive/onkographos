# Hardcoded Transfer Function

Also called: Fixed RGBA Array, Static 1D Lookup Table, 1D Transfer Function, 1D data-based TF, 1dTF

## Overview
The Hardcoded Transfer Function (HTF) is a classification method that maps scalar volume data values directly to color and opacity using a static, pre-defined lookup table [S1, unofficial][S2][S4]. It solves the problem of visualizing scalar fields by assigning RGBA values based solely on intensity, serving as the baseline reference for transfer function complexity [S2]. It is distinguished from its siblings by its complete lack of parameterization or runtime generation; the color/opacity mapping is fixed in memory, requiring no user interaction or mathematical derivation during rendering [S2]. Unlike [2D Transfer Function Volume Rendering](1295-2d-transfer-function-volume-rendering.md), it cannot distinguish between features with identical scalar values but different structural contexts [S2][S4].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar volume dataset with values $v \in [x_{min}, x_{max}]$.
*   **Output:** A 4-component vector $(R, G, B, \alpha)$ for each sample point.

**Data Structures:**
The transfer function is stored as a static array of 4-component vectors.
*   **Layout:** An array of `float` or `vec4` elements.
*   **Size:** Commonly 256 entries (mapping 8-bit indices 0–255) or 4096 entries (for higher precision or 12-bit internal representations) [S1, unofficial][S3, unofficial].
*   **Content:** Each entry $i$ contains $(R_i, G_i, B_i, \alpha_i)$ where components are in $[0, 1]$ [S1, unofficial][S3, unofficial]. In some hardware implementations (e.g., VolumePro), RGB components are 8-bit repeating-fraction numbers and alpha is a 12-bit repeating-fraction number, totaling 36 bits per entry [S3, unofficial].
*   **GPU Storage:** Uploaded to GPU memory as a 1D texture (for hardware interpolation) or a raw device array/SSBO (for direct lookup) [S5, unofficial].

**Steps:**
1.  **Normalization:** Map the input scalar value $v$ to a normalized domain $x \in [0, 1]$:
    $$ x = \frac{v - x_{min}}{x_{max} - x_{min}} $$
    [S2][S3, unofficial][S4][S5, S6, S8, unofficial]
2.  **Index Calculation:** Compute the integer index $idx$ into the lookup table of size $N$:
    $$ idx = \text{clamp}(\lfloor x \cdot (N-1) \rfloor, 0, N-1) $$
    [S3, S5, S8, unofficial]
    *Note: If using hardware texture sampling, the texture object is configured with `normalizedCoords = 1` and `addressMode = Clamp` to handle this automatically [S5, unofficial].*
3.  **Lookup:** Retrieve the RGBA value from the table.
    *   **Direct Lookup:** Return `table[idx]` [S5, unofficial].
    *   **Interpolated Lookup:** If the table is stored as a linear-filtered texture, sample at coordinate $x$ to get a weighted average of adjacent entries [S5, unofficial].
4.  **Output:** Return the retrieved $(R, G, B, \alpha)$.

**Pseudocode (Direct Lookup):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hardcoded_transfer_function_lookup(value, x_min, x_max, table, N):
    """
    value: float, scalar input from volume
    x_min: float, minimum scalar value in volume
    x_max: float, maximum scalar value in volume
    table: list of vec4, static RGBA lookup table of size N
    N: int, length of table
    """
    # 1. Normalize
    if x_max == x_min:
        x = 0.0
    else:
        x = (value - x_min) / (x_max - x_min)
    
    # 2. Clamp to [0, 1]
    x = max(0.0, min(1.0, x))
    
    # 3. Calculate Index
    idx = int(x * (N - 1))
    idx = max(0, min(N - 1, idx))
    
    # 4. Lookup and Return
    return table[idx]
```

**Parameters:**
*   **Table Content:** The specific RGBA values are hardcoded. Typical pitfalls include opacity values being mostly 0.0 or 0.1, with limited dynamic range [S1, unofficial].
*   **Domain:** $x_{min}$ and $x_{max}$ define the scalar range mapped to the table [S5, unofficial].

**Edge Cases and Pitfalls:**
*   **Identical Values:** The method cannot differentiate between samples with the same scalar value, even if they represent different materials or boundaries [S2][S4].
*   **Precision:** Using a small table size (e.g., 256) may cause banding if the input data has higher precision [S3, unofficial].
*   **Out-of-Bounds:** Values outside $[x_{min}, x_{max}]$ must be clamped to prevent memory errors [S2][S3, unofficial][S4][S5, S6, S7, S8, unofficial].

## Subtopics

### Fire Color Palette
A hardcoded 1D color mapping function that maps a normalized value $d \in [0, 1]$ to a gradient from gray to yellow [S7, unofficial]. It is used for aesthetic or specific thermal-like visualization [S7, unofficial].
*   **Definition:** Piecewise linear interpolation between five color stops: `gray` (0.4, 0.4, 0.4), `darkgray` (0.2, 0.2, 0.2), `red` (1.0, 0.0, 0.0), `orange` (1.0, 0.6, 0.0), and `yellow` (1.7, 1.3, 1.0) [S7, unofficial]. Note that yellow components can exceed 1.0 [S7, unofficial].
*   **Steps:**
    1.  Clamp input $x$ to $[0.0, 1.0]$.
    2.  If $x < 0.25$, lerp `gray` to `darkgray` using $t = x \cdot 4.0$.
    3.  If $0.25 \le x < 0.5$, lerp `darkgray` to `red` using $t = x \cdot 4.0 - 1.0$.
    4.  If $0.5 \le x < 0.75$, lerp `red` to `orange` using $t = x \cdot 4.0 - 2.0$.
    5.  If $x \ge 0.75$, lerp `orange` to `yellow` using $t = x \cdot 4.0 - 3.0$.
*   **When to choose:** When a specific "fire" aesthetic is required without runtime parameterization [S7, unofficial].

### Exponential Color Transfer Function
A hardcoded method that maps intensity to color and alpha using an exponential alpha curve and linear color interpolation [S8, unofficial].
*   **Definition:** Maps intensity to a grayscale color (black to white) with exponential opacity decay [S8, unofficial].
*   **Steps:**
    1.  Define `high = vec3(1.0, 1.0, 1.0)` and `low = vec3(0.0, 0.0, 0.0)`.
    2.  Compute color: `color = intensity * high + (1.0 - intensity) * low`.
    3.  Compute alpha: $\alpha = \frac{e^{intensity} - 1}{e^1 - 1}$.
    4.  Return `vec4(color, alpha)`.
*   **When to choose:** When a simple, non-parameterized exponential fade-out for opacity is desired for single-channel data [S8, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample. The lookup is a direct array access or texture fetch [S2].
*   **Memory Complexity:** $O(N)$, where $N$ is the table size (e.g., 256 or 4096 entries) [S2][S3, unofficial]. This is the lowest memory footprint among transfer function types [S2].
*   **Performance:** Lowest computation power requirement [S2].
*   **Strengths:** High safety against misinterpretation errors due to simplicity; little user interaction required [S2].
*   **Limitations:** Cannot distinguish features with identical scalar values but different contexts (e.g., boundaries, textures) [S2][S4]. Cannot handle multi-channel data natively [S4].
*   **When to use:** Quick visualization, scalar-only data where intensity is the sole distinguishing feature, or as a baseline for comparison [S2].
*   **When not to use:** When differentiating materials with similar intensities is required, or when multi-channel data is present [S2][S4].

## Background
The 1D data-based transfer function (1dTF) is the foundational type of transfer function in volume rendering [S2]. It was formally compared against other TF types (gradient-based, curvature-based, etc.) in a survey by [S2] (2010), establishing it as the reference point for simplicity and low resource usage. Early hardware implementations, such as the VolumePro system, utilized large pre-computed 1D LUTs (4096 entries) for real-time ray-casting [S3, unofficial]. The concept is widely cited in educational and open-source implementations as the starting point for transfer function design [S1, unofficial][S5, unofficial][S6, unofficial][S8, unofficial].

## Sources
[S1] extrovert7986/Scientific-visualization-Ray-casting-volume-rendering, https://github.com/extrovert7986/Scientific-visualization-Ray-casting-volume-rendering, fetched 2026-10-26 (unofficial)
[S2] A Survey of Transfer Functions Suitable for Volume Rendering, https://doi.org/10.2312/vg/vg10/077-083, fetched 2026-10-26
[S3] [PDF] The VolumePro Real-Time Ray-Casting System, https://www.merl.com/publications/docs/TR99-19.pdf, fetched 2026-10-26 (unofficial)
[S4] PDF Dimensionality reduction on multi-dimensional transfer functions for ..., https://cseweb.ucsd.edu/~hskim/files/IVS_ivs20106.pdf, fetched 2026-10-27
[S5] Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, https://github.com/Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, fetched 2026-10-27 (unofficial)
[S6] GPU-accelerated single-pass volumetric raycasting in Qt ..., https://martinopilia.com/posts/2018/09/17/volume-raycasting.html, fetched 2026-10-27 (unofficial)
[S7] OmerNYU/Kaboom_Diffusion, https://github.com/OmerNYU/Kaboom_Diffusion, fetched 2026-10-26 (unofficial)
[S8] GitHub - m-pilia/volume-raycasting: Simple visualiser based on GPU ..., https://github.com/m-pilia/volume-raycasting, fetched 2026-10-27 (unofficial)
