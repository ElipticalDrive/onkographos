# Tent Alpha Transfer Function

Also called: FluoRender Transfer Function, Fluorescence Transfer Function, tent-based transfer function

## Overview
The Tent Alpha Transfer Function is a hardcoded 1D RGBA transfer texture generation method for fluorescence microscopy data that maps 8-bit scalar values to color and opacity using a tent-shaped alpha curve and gamma corrections [S1, unofficial]. It provides a parametric opacity profile centered on a user-defined peak value [S1, unofficial]. It builds on the [Hardcoded Transfer Function](1098-hardcoded-transfer-function.md) by defining the color and opacity as a static float array generated from a closed-form mathematical profile rather than a manually edited lookup table [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**: `alpha0` (alpha at data value 0, typical 0), `peak` (data value where alpha reaches maximum), `dataGamma` (exponent controlling the rise/fall shape, typical 0.5), `alpha1` (alpha at data value 255, typical 255), and `colorStr` (base hex color string applied to all data values) [S1, unofficial]. `finalGamma` is an optional exponent applied later during ray casting to the accumulated color [S1, unofficial].
*   **Output**: A 256×1 RGBA texture where each entry corresponds to a data value 0–255 [S1, unofficial].

**Data Structures**
*   **Transfer Texture**: A 1D texture with width 256, height 1, format RGBA, and wrapping set to ClampToEdge for both S and T axes [S1, unofficial].
*   **Data Array**: A `Uint8Array` of length $4 \times 256 = 1024$ [S1, unofficial]. Indices $4i$ through $4i+3$ store R, G, B, A for data value $i$ [S1, unofficial]. Values are clamped to [0, 255] [S1, unofficial].

**Parameters and Typical Values**
*   `alpha0`: 0 [S1, unofficial].
*   `alpha1`: 255 [S1, unofficial].
*   `dataGamma`: 0.5 [S1, unofficial].
*   `peak`: User-defined integer determining the tent center [S1, unofficial].

**Algorithm Steps**
1.  **Initialize Color**: Parse `colorStr` into RGB components $(r, g, b)$ as integers in [0, 255] [S1, unofficial].
2.  **Initialize Texture Data**: Create `data` as `Uint8Array(1024)` [S1, unofficial].
3.  **Iterate Data Values**: Loop $i$ from 0 to 255 (inclusive) [S1, unofficial].
4.  **Calculate Tent Geometry**:
    *   Calculate distance $d$ to the nearest edge relative to the peak:
        $$
        d = \begin{cases} 
        \text{peak} & \text{if } i < \text{peak} \\
        256 - \text{peak} - 1 & \text{if } i \ge \text{peak}
        \end{cases}
        $$
    *   Calculate normalized position $x$:
        $$
        x = \begin{cases} 
        i & \text{if } i < \text{peak} \\
        256 - i - 1 & \text{if } i \ge \text{peak}
        \end{cases}
        $$
    *   Calculate tent height $y$ (normalized to [0, 1]):
        $$
        y = \frac{x}{d}
        $$
    *   *Note*: If $d=0$ (i.e., `peak`=0 and $i=0$), $y$ becomes undefined/Infinity [S1, unofficial].
5.  **Apply Data Gamma**:
    $$
    y_{\text{Gamma}} = y^{\frac{1.0}{\text{dataGamma}}}
    $$
6.  **Calculate Alpha**:
    *   Determine base alpha $\alpha_{\text{base}}$:
        $$
        \alpha_{\text{base}} = \begin{cases} 
        \text{alpha0} & \text{if } i < \text{peak} \\
        \text{alpha1} & \text{if } i \ge \text{peak}
        \end{cases}
        $$
    *   Update alpha using the tent shape:
        $$
        \alpha = \alpha_{\text{base}} + y_{\text{Gamma}} \times (255 - \alpha_{\text{base}})
        $$
    *   Clamp $\alpha$ to [0, 255] using rounding:
        $$
        \alpha = \text{clamp}(\text{round}(\alpha), 0, 255)
        $$
7.  **Calculate Final RGBA**:
    *   Calculate extra alpha factor:
        $$
        \alpha_{\text{norm}} = \frac{\alpha}{255.0}
        $$
    *   Set RGBA values in `data` at index $4i$:
        $$
        \begin{aligned}
        R &= \text{round}(r \times \alpha_{\text{norm}}) \\
        G &= \text{round}(g \times \alpha_{\text{norm}}) \\
        B &= \text{round}(b \times \alpha_{\text{norm}}) \\
        A &= \alpha
        \end{aligned}
        $$
    *   The color channels are modulated by alpha, but the alpha channel itself is stored directly [S1, unofficial].
8.  **Finalize Texture**: Create the texture object from `data` with width 256, height 1, format RGBA, and mark as needing update [S1, unofficial]. Optionally attach `watermark` (the `peak` value) to `texture.userData.watermark` for verification [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def make_fluo_transfer_tex(alpha0, peak, data_gamma, alpha1, color_str):
    """
    Generates a 256x1 RGBA transfer texture for fluorescence data.
    """
    # 1. Parse Color: Convert hex string to RGB integers [0, 255]
    r = int(color_str[1:3], 16)
    g = int(color_str[3:5], 16)
    b = int(color_str[5:7], 16)

    # 2. Initialize Texture Data
    data = bytearray(1024)  # 256 * 4

    # 3. Iterate Data Values
    for i in range(256):
        # 4. Calculate Tent Geometry
        if i < peak:
            d = peak
            x = i
        else:
            d = 256 - peak - 1
            x = 256 - i - 1

        # Guard against division by zero; source notes d=0 causes Infinity/NaN [S1, unofficial]
        if d == 0:
            y = 0.0
        else:
            y = x / d

        # 5. Apply Data Gamma
        y_gamma = y ** (1.0 / data_gamma)

        # 6. Calculate Alpha
        if i < peak:
            alpha_base = alpha0
        else:
            alpha_base = alpha1

        alpha = alpha_base + y_gamma * (255 - alpha_base)
        alpha = max(0, min(255, round(alpha)))

        # 7. Calculate Final RGBA
        extra_alpha = alpha / 255.0

        idx = i * 4
        data[idx] = max(0, min(255, round(r * extra_alpha)))
        data[idx + 1] = max(0, min(255, round(g * extra_alpha)))
        data[idx + 2] = max(0, min(255, round(b * extra_alpha)))
        data[idx + 3] = alpha

    # 8. Finalize Texture
    return {
        'data': bytes(data),
        'width': 256,
        'height': 1,
        'format': 'RGBA',
        'wrapping': 'ClampToEdge'
    }
```

**Edge Cases and Pitfalls**
*   **Division by Zero**: If `peak` is 0, $d=0$ when $i=0$. If `peak` is 255, $d=0$ when $i=255$ (since $256-255-1=0$) [S1, unofficial]. The source code does not explicitly check for $d=0$, leading to `Infinity` or `NaN` in $y$ calculation [S1, unofficial].
*   **Gamma Interpretation**: `dataGamma` is applied to the tent height before alpha blending, while `finalGamma` is applied to the accumulated color after ray casting/compositing [S1, unofficial].
*   **Alpha Modulation**: The color channels are multiplied by `alpha/255`, storing premultiplied alpha values [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(1)$ for texture generation (fixed 256 iterations) and $O(N)$ for ray casting, where $N$ is the number of samples along the ray [G].
*   **Memory Complexity**: $O(1)$ for the 1024-byte transfer texture [G].
*   **Performance**: `dataGamma` < 1 reduces visibility of low data values during ray casting to prevent them from overwhelming significant features [S1, unofficial].
*   **Strengths**: Not given in the sources.
*   **Limitations**: Not given in the sources.
*   **When to Use**: Not given in the sources.
*   **When Not to Use**: Not given in the sources.

## Background
The method is based on FluoRender by Wan et al., published in the Proceedings of the 2012 IEEE Pacific Visualization Symposium [S1, unofficial]. It leverages 2D image space methods for 3D and 4D confocal microscopy data visualization in neurobiology research [S1, unofficial].

## Sources
[S1] JaneliaSciComp/web-vol-viewer, https://github.com/JaneliaSciComp/web-vol-viewer, fetched 2026-10-26 (unofficial)
