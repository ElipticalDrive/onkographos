# ACES Filmic Tone Mapping

Also called: ACES Filmic, ACES Tonemapping, ACES

## Overview
ACES Filmic Tone Mapping is a filmic tone mapping function that compresses HDR color into $[0, 1]$ using a characteristic shoulder curve to prevent highlight clipping [S3, unofficial]. It differs from simple Reinhard tonemapping by providing a non-linear highlight roll-off rather than a basic division [S1, unofficial]. It builds on [Cloud Compositing & Tone Mapping](3628-cloud-compositing-tone-mapping.md).

## How it works
**Inputs and Outputs:**
- **Input:** A linear HDR color vector $\mathbf{c} = (r, g, b)$, where each component represents physical radiance or scene-referred luminance [G].
- **Output:** A linear LDR color vector $\mathbf{c}' = (r', g', b')$ with each component clamped to the range $[0.0, 1.0]$ [S1, unofficial][S3, unofficial].

**Data Structures:**
No complex data structures are required. The method operates per-pixel on scalar color components [G].

**Parameters:**
The curve is defined by five constants. These values are fixed for the standard ACES approximation [S1, unofficial][S3, unofficial][S4, unofficial]:
- $a = 2.51$
- $b = 0.03$
- $c = 2.43$
- $d = 0.59$
- $e = 0.14$

**Formula:**
The tone mapping function is applied independently to each color channel $x \in \{r, g, b\}$:

$$
\text{toneMapACES}(x) = \text{clamp}\left( \frac{x (a x + b)}{x (c x + d) + e}, 0.0, 1.0 \right)
$$

Where:
- $x$ is the input linear color value for a single channel [G].
- $\text{clamp}(v, \text{min}, \text{max})$ restricts $v$ to the interval $[\text{min}, \text{max}]$ [G].
- The numerator $x(ax+b)$ provides a linear response for low values and quadratic growth for mid-values [G].
- The denominator $x(cx+d)+e$ ensures the curve asymptotically approaches a finite limit, creating the shoulder that compresses highlights [G].

**Steps:**
1. For each pixel in the image, retrieve the linear RGB color values $(r, g, b)$ from the rendering buffer [G].
2. For each color channel $x$ in $(r, g, b)$:
   a. Calculate the numerator: $num = x \cdot (a \cdot x + b)$.
   b. Calculate the denominator: $den = x \cdot (c \cdot x + d) + e$.
   c. Compute the ratio: $y = num / den$.
   d. Clamp the result: $x' = \max(0.0, \min(1.0, y))$.
3. Construct the output color vector $\mathbf{c}' = (r', g', b')$ [G].
4. [G] The output $\mathbf{c}'$ is in linear LDR space. It is typically followed by gamma correction with $\gamma=2.2$ if the target display format requires it [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tone_map_aces(color):
    """
    Applies the ACES Filmic tone mapping curve to a linear RGB color.
    
    Args:
        color: A tuple (r, g, b) of linear HDR values.
        
    Returns:
        A tuple (r', g', b') of LDR values in [0.0, 1.0].
    """
    a, b = 2.51, 0.03
    c, d, e = 2.43, 0.59, 0.14
    r, g, b_val = color
    
    def apply_channel(x):
        num = x * (a * x + b)
        den = x * (c * x + d) + e
        y = num / den
        if y < 0.0:
            return 0.0
        if y > 1.0:
            return 1.0
        return y

    return (apply_channel(r), apply_channel(g), apply_channel(b_val))
```

**Edge Cases and Pitfalls:**
- **Double Tonemapping:** If the rendering engine or pipeline already applies ACES tonemapping (e.g., Unreal Engine's default post-process stack), applying this function again will distort colors. In such cases, disable this step [S2, unofficial].
- **Identity Function Limitation:** The Narkowicz ACES approximation cannot represent the identity function. If the ground truth data is true linear HDR without tonemapping, do not use learnable ACES variants, as they will fail to model the identity mapping [S2, unofficial].
- **Precision:** For very high HDR values, the curve asymptotically approaches $a/c \approx 1.033$, which is then clamped to $1.0$ [G]. This is the intended behavior for highlight compression.

## Subtopics

### Learnable ACES
A variant where the four coefficients $a, b, c, d$ are treated as learnable parameters during optimization, while $e$ remains fixed at $0.14$ [S2, unofficial]. This adaptation allows the tone mapping curve to fit the specific display characteristics of the ground truth data or other filmic engines more closely [S2, unofficial]. The learned coefficients are saved in `tonemap.json` in the same directory as the asset data [S2, unofficial]. It is chosen when the standard ACES curve does not adequately match the target display's response or when serving as a fallback for other filmic engines [S2, unofficial]. It is not suitable if the input is true linear HDR requiring an identity mapping [S2, unofficial].

### Log-Contrast and Saturation Tone Curve
A post-processing adjustment that applies log contrast centered around mid-grey values and includes saturation controls [S5, unofficial]. Unlike ACES Filmic, which focuses on dynamic range compression and highlight preservation, this method primarily adjusts the perceptual contrast and color vibrancy of the image [S5, unofficial]. It is applied to the scene-linear image before tone mapping in some pipelines (e.g., Octane's imager / KeyShot image styles) to enhance visual appeal [S5, unofficial]. It is chosen when fine-tuning the mid-tone contrast and color richness is required, rather than solving HDR-to-LDR mapping issues [S5, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per pixel. The calculation involves a fixed number of arithmetic operations (multiplications, additions, one division) and a clamp [G].
- **Memory Complexity:** $O(1)$. No additional memory is required beyond the input and output color buffers [G].
- **Performance:** Measured performance is negligible in terms of computational cost compared to volume ray casting or compositing steps [G].
- **Strengths:** Provides a natural, filmic look with smooth highlight roll-off [S3, unofficial]. Prevents harsh clipping of bright highlights [S3, unofficial]. Simple to implement and computationally inexpensive [G]. Matches the "Ground Truth" display space used in some cloud rendering datasets [S2, unofficial].
- **Limitations:** Not suitable for scenes where the input is already LDR or where a linear pass-through is required [S2, unofficial]. Requires subsequent gamma correction for display [S1, unofficial].
- **When to use:** Use when rendering HDR physical shading outputs that need to be displayed on standard LDR monitors [G].
- **When not to use:** Do not use if the pipeline already applies tonemapping (risk of double tonemapping) [S2, unofficial], or if the input data is true linear HDR and requires an identity mapping [S2, unofficial].

## Background
The ACES Filmic tone mapping curve is an approximation of the Academy Color Encoding System (ACES) look, originally described by Narkowicz in 2015 [S1, unofficial]. It was designed to provide a standardized, filmic response curve for digital intermediate workflows, mimicking the characteristic highlight roll-off of photographic film [G]. In computer graphics and volume rendering, it has been adopted as a standard post-processing step to ensure consistent output from physically based renderers [S2, unofficial][S3, unofficial]. It is widely used in real-time engines (like Unreal Engine) and offline renderers (like Photon Engine) as a default or configurable tone mapping option [S2, unofficial][S5, unofficial].

## Sources
[S1] jamestkiernan/webgpu-voxel-raymarching, https://github.com/jamestkiernan/webgpu-voxel-raymarching, fetched 2026-10-24 (unofficial)
[S2] hsiang0117/4DGS-Volume-Cloud, https://github.com/hsiang0117/4DGS-Volume-Cloud, fetched 2026-10-26 (unofficial)
[S3] PIGJET/neon-raymarcher, https://github.com/PIGJET/neon-raymarcher, fetched 2026-10-27 (unofficial)
[S4] wlim33/vox-ocean, https://github.com/wlim33/vox-ocean, fetched 2026-10-27 (unofficial)
[S5] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
