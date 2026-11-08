# Colormap LUT Generation

Also called: Named Colormap LUT Generation, TransferFunction Colormap Generation

## Overview
Colormap LUT Generation is the procedural creation of a 1D lookup table (typically 256 entries of RGBA) from a named colormap definition or a set of key-value pairs, serving as the color mapping stage in volume rendering classification [S1, S2, S3, unofficial]. It differs from [Parametric Function Generation](5349-parametric-function-generation.md) by using predefined discrete color gradients rather than continuous mathematical functions, and from [1D Transfer Function Generation and Sampling](2689-1d-transfer-function-generation-and-sampling.md) by focusing on the static texture creation rather than the runtime interpolation of arbitrary user-defined transfer functions.

## How it works
**Inputs and Outputs:**
*   **Input:** A colormap identifier (e.g., 'viridis', 'hot', 'gray') [S2, S3, unofficial] or a list of key-value pairs (position, RGBA) [S1, unofficial].
*   **Output:** A 1D texture or array of 256 (or $N$) RGBA values, where each component is typically an 8-bit unsigned integer (uint8) [S1, S2, S3, unofficial].

**Data Structures:**
*   **LUT Array:** A contiguous array of size $N$ (default 256) [S3, unofficial]. Each element is a 4-component vector (R, G, B, A) [S1, S3, unofficial].
*   **Texture Structure:** In GPU implementations, this is often wrapped in a structure containing dimensions (`size.x`, `size.y`) and a `Uint8Array` of RGBA data [S1, unofficial].

**Steps:**
1.  **Initialize LUT:** Create an empty array of size $N$ (default 256) [S3, unofficial]. Clear any existing LUT data [S2, unofficial].
2.  **Select Colormap:** Identify the target colormap by name (e.g., 'inferno', 'jet', 'viridis') [S2, S3, unofficial] or use a procedural definition [S1, unofficial].
3.  **Generate Colors:** For each index $i$ from 0 to $N-1$:
    *   Calculate normalized position $t = i / (N - 1)$ [S1, S3, unofficial]. Note: Some implementations use $t = i / N$ [S3, unofficial].
    *   Compute RGB values based on the colormap algorithm:
        *   **Gray:** $R=G=B = \lfloor i \rfloor$, $A=255$ [S1, unofficial].
        *   **Hot:** $R = \min(255, i \times 3)$, $G = \min(255, \max(0, (i - 85) \times 3))$, $B = \min(255, \max(0, (i - 170) \times 3))$, $A=255$ [S1, unofficial].
        *   **Viridis:** $t = i / 255$, $R = \lfloor 68 + 187t \rfloor$, $G = \lfloor 1 + 231t \rfloor$, $B = \lfloor 84 - 82t \rfloor$, $A=255$ [S1, unofficial].
        *   **Red:** $R=255, G=0, B=0, A=255$ for all entries [S1, unofficial].
        *   **Generic/Named:** Use a library function (e.g., `cmap.Colormap(name).lut()`) [S2, unofficial] or `tinycolormap::GetColor(f, type)` [S3, unofficial] to retrieve RGB, then scale to uint8 [S2, unofficial].
    *   Store the resulting RGBA vector in the LUT at index $i$ [S3, unofficial].
4.  **Apply Opacity Modifiers (Optional):**
    *   **Fading:** Clone the LUT. For each $i$, set $A_i = \lfloor (i / (N - 1)) \times 255 \rfloor$ [S1, unofficial].
    *   **Clamping:** Clone the LUT. For each $i$, if $minAlpha < i < maxAlpha$, keep original $A$; otherwise set $A=0$ [S1, unofficial].
5.  **Upload/Export:** Convert the array to the target format (e.g., `Uint8Array`) [S1, unofficial], reshape if necessary for image export (e.g., $1 \times N \times 3$) [S2, unofficial], and upload to GPU memory [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_colormap_lut(name: str, n_bins: int = 256) -> list:
    """
    Generate a 1D LUT for a named colormap.
    Returns a list of 4-tuples (R, G, B, A) as integers 0-255.
    """
    lut = []
    
    # Helper for standard named colormaps defined in sources
    def get_color_from_name(name, i, n):
        t = i / (n - 1)
        if name == 'gray':
            return (i, i, i, 255)
        elif name == 'hot':
            r = min(255, i * 3)
            g = min(255, max(0, (i - 85) * 3))
            b = min(255, max(0, (i - 170) * 3))
            return (r, g, b, 255)
        elif name == 'viridis':
            # Formula from [S1, unofficial]
            r = int(68 + 187 * t)
            g = int(1 + 231 * t)
            b = int(84 - 82 * t)
            return (r, g, b, 255)
        elif name == 'red':
            return (255, 0, 0, 255)
        else:
            # Fallback for other named colormaps (e.g., jet, inferno)
            # In practice, this would call a library like cmap or tinycolormap
            # Here we simulate a linear gradient for unknown names as a placeholder
            # [G] Generic linear interpolation for unknown names to ensure completeness
            val = int(255 * t)
            return (val, val, val, 255)

    for i in range(n_bins):
        r, g, b, a = get_color_from_name(name, i, n_bins)
        lut.append((r, g, b, a))
        
    return lut

def apply_opacity_fade(lut: list) -> list:
    """
    Apply opacity fading to an existing LUT.
    Alpha becomes proportional to the index.
    """
    n = len(lut)
    new_lut = []
    for i, (r, g, b, a) in enumerate(lut):
        new_a = int((i / (n - 1)) * 255)
        new_lut.append((r, g, b, new_a))
    return new_lut

def apply_opacity_clamp(lut: list, min_alpha_idx: int, max_alpha_idx: int) -> list:
    """
    Apply opacity clamping.
    Alpha is 0 outside the range (min_alpha_idx, max_alpha_idx).
    """
    new_lut = []
    for i, (r, g, b, a) in enumerate(lut):
        if min_alpha_idx < i < max_alpha_idx:
            new_lut.append((r, g, b, a))
        else:
            new_lut.append((r, g, b, 0))
    return new_lut
```

**Parameters:**
*   `name`: String identifier for the colormap (e.g., 'inferno', 'jet', 'viridis', 'hot', 'gray', 'red') [S2, S3, unofficial]. Default is often 'inferno' [S2, unofficial].
*   `n_bins`: Number of entries in the LUT. Default is 256 [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Precision:** When calculating $t$, using $i/(N-1)$ ensures the last entry maps to 1.0, while $i/N$ leaves the last entry slightly short. Sources vary in their usage [S1, S3, unofficial].
*   **Alpha Interpretation:** Some colormaps (like 'hot' in [S1, unofficial]) have constant alpha, while others might require explicit opacity mapping via fading or clamping [S1, unofficial].
*   **Library Dependency:** Generating non-standard colormaps (e.g., 'turbo', 'cividis') requires external libraries like `cmap` [S2, unofficial] or `tinycolormap` [S3, unofficial], which may have different normalization behaviors.

## Subtopics

### Named Colormap Selection
Choosing from standard perceptually uniform or sequential colormaps (Viridis, Jet, etc.). Choose when visual consistency and perceptual accuracy are prioritized over custom data mapping. See [Parametric Function Generation](5349-parametric-function-generation.md) for mathematical alternatives.

### Opacity Modulation
Applying fading or clamping to the alpha channel. Choose when depth cueing or region isolation is needed. See [Transfer Function Editor (Alpha Pins)](6772-transfer-function-editor-alpha-pins.md) for interactive control.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of bins (typically 256). The generation is extremely fast and usually performed once at initialization [S1, S2, S3, unofficial].
*   **Memory Complexity:** $O(N)$ bytes (4 bytes per RGBA entry). For $N=256$, this is 1 KB [S1, unofficial].
*   **Performance:** Negligible runtime cost. The LUT is generated once and uploaded to GPU texture memory [S3, unofficial].
*   **Strengths:** Simple to implement; provides standard, visually pleasing color maps; fast generation.
*   **Limitations:** Fixed color gradients; cannot adapt to dynamic data ranges without regeneration; less flexible than parametric transfer functions for complex classification tasks [S1, S3, unofficial].
*   **When to Use:** Standard volume visualization where predefined color maps are sufficient.
*   **When Not to Use:** When custom, data-dependent color mapping is required (use [Parametric Function Generation](5349-parametric-function-generation.md) or [1D Transfer Function Generation and Sampling](2689-1d-transfer-function-generation-and-sampling.md)).

## Background
Colormap LUT generation is a fundamental step in volume rendering visualization, originating from the need to map scalar data to visible colors. Early implementations used simple procedural gradients (gray, hot) [S1, unofficial]. Modern implementations leverage libraries like `cmap` [S2, unofficial] and `tinycolormap` [S3, unofficial] to support perceptually uniform colormaps (Viridis, Inferno) introduced in the 2010s to address issues with traditional maps like Jet. It is widely used in scientific visualization tools and real-time volume renderers [S1, S2, S3, unofficial].

## Sources
[S1] jaanonim/volume-rendering, https://github.com/jaanonim/volume-rendering, fetched 2026-10-26 (unofficial)
[S2] PeterC-DLS/webgl_volume_demo, https://github.com/PeterC-DLS/webgl_volume_demo, fetched 2026-10-27 (unofficial)
[S3] nihofm/volren: Simple OpenGL-based renderer for ..., https://github.com/nihofm/volren, fetched 2026-10-29 (unofficial)
