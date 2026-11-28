# Tonemapping and Gamma Correction

Also called: Color Clamping, Exposure and Gamma Correction, Final image processing, Gamma Correction & PPM Output, PPM Writer, tonemap

## Overview
Tonemapping and Gamma Correction compresses the high dynamic range (HDR) radiance values produced by volume rendering into a low dynamic range (LDR) range suitable for display devices, while correcting for the non-linear response of monitors [S2][S3, unofficial]. It differs from implicit tone mapping methods by explicitly separating exposure control, luminance-based desaturation, and gamma encoding, allowing for precise artistic control over shadow depth and color saturation [S1, S3, unofficial].

## How it works
**Inputs and Outputs:**
- Input: HDR fragment color $C_{\text{in}}$ (typically `vec3` in `[0, \infty)`), exposure constant $E$, gamma exponent $\gamma$, samples per pixel $S$, and optional black point offset vector $\mathbf{b}$ [S2][S3, unofficial].
- Output: LDR display color $C_{\text{out}}$ (typically `vec3` in `[0, 1]` or `[0, 255]`) [S2][S3, unofficial].

**Data Structures:**
- `color` texture: `GL_RGBA32F` format, bound to image unit 0 with `GL_READ_WRITE` access [S4, unofficial].
- Uniforms: `resolution` (vec2), `exposure` (float), `gamma` (float) [S4, unofficial].

**Steps:**
1. **Exposure Adjustment:** Apply exponential tonemapping to compress HDR values [S2][S3, unofficial]. The formula is:
   $$ I_F = 1 - \exp(-E \cdot I_F) $$
   where $E$ is the exposure constant (e.g., 1.35) [S2][S3, unofficial].
2. **Luminance-Based Desaturation:** To counteract the desaturation of bright colors caused by exponential tonemapping, mix the original color with a luminance-only version [S3, unofficial].
   - Calculate luminance: $L = \text{dot}(C, \mathbf{w})$, where $\mathbf{w} = (0.2126, 0.7152, 0.0722)$ [S3, unofficial].
   - Mix: $C = \text{clamp}(\text{mix}(\mathbf{L}, C, \alpha), 0.0, 1.0)$, where $\mathbf{L} = (L, L, L)$ and $\alpha$ is the chroma mix factor (e.g., 1.32) [S3, unofficial].
3. **Black Level Offset (Optional):** Subtract channel-specific black points to restore deep shadows and prevent haze caused by gamma lifting [S5, unofficial].
   - $C = \text{max}(C - \mathbf{b}, \mathbf{0})$, where $\mathbf{b} = (0.0175, 0.0160, 0.0060)$ for R, G, B respectively [S5, unofficial].
4. **Gamma Correction:** Apply gamma encoding to match the monitor's response [S3, unofficial].
   - $C = C^{1/\gamma}$, where $\gamma$ is typically 2.2 [S3, unofficial].
5. **Scaling and Clamping (PPM Output Variant):** If outputting to PPM format, scale to 8-bit integers [S1, unofficial].
   - $C_{\text{out}} = \text{clamp}\left(255 \sqrt{\frac{C_{\text{in}}}{S}}, 0, 255\right)$, where $S$ is `samplesPerPixel` [S1, unofficial].
   - Cast to integer [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tonemap_and_gamma_correction(color: vec3, exposure: float, gamma: float, 
                                 chroma_mix_factor: float = 1.32, 
                                 black_point: vec3 = vec3(0.0, 0.0, 0.0)) -> vec3:
    # Step 1: Exposure Adjustment
    color = 1.0 - exp(-exposure * color)
    
    # Step 2: Luminance-Based Desaturation
    luminance = dot(color, vec3(0.2126, 0.7152, 0.0722))
    color = clamp(mix(vec3(luminance), color, chroma_mix_factor), 0.0, 1.0)
    
    # Step 3: Black Level Offset
    color = max(color - black_point, vec3(0.0))
    
    # Step 4: Gamma Correction
    color = pow(color, 1.0 / gamma)
    
    return color

def tonemap_ppm_output(color: vec3, samples_per_pixel: float) -> vec3:
    # PPM output variant: scale and clamp to 8-bit range
    scaled = 255.0 * sqrt(color / samples_per_pixel)
    return clamp(scaled, 0.0, 255.0)
```

**Parameters:**
- `exposure`: Controls brightness compression. Typical value: 1.35 [S3, unofficial].
- `gamma`: Monitor response exponent. Typical value: 2.2 [S3, unofficial].
- `chroma_mix_factor`: Weight for chroma preservation. Typical value: 1.32 [S3, unofficial].
- `black_point`: Channel-specific offset. Typical value: `(0.0175, 0.0160, 0.0060)` [S5, unofficial].
- `samples_per_pixel`: Used in PPM scaling. Typical value: inverse of the sample count (i.e., `1.0 / samplesPerPixel`) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Gamma Lifting Haze:** Gamma correction ($C^{1/2.2}$) lifts near-black values significantly. A linear value of 0.001 becomes visible haze. This causes washed-out shadows and incorrect color balance (e.g., teal instead of blue-violet) if not corrected with black point subtraction [S5, unofficial].
- **Desaturation of Bright Colors:** Exponential tonemapping compresses the dominant channel hardest, leading to desaturation of bright colors (e.g., orange drifting to tan). Chroma-preserving mixing is required to restore saturation [S3, unofficial].

## Subtopics

### [Explicit Tone Mapping](11070-explicit-tone-mapping.md)
Explicit Tone Mapping converts HDR radiance to LDR by modeling the physical image acquisition process of a digital camera, parameterizing white balance and Camera Response Function (CRF) as distinct, differentiable stages for self-calibration during HDR radiance field optimization. It differs from implicit methods by allowing direct optimization of camera parameters. Choose this when integrating tone mapping into differentiable rendering pipelines for neural radiance fields.

### Channel-wise Black Point Correction
A post-processing step to correct color purity in shadows by subtracting a channel-specific black point vector $(0.0175, 0.0160, 0.0060)$ after gamma lifting [S5, unofficial]. It prevents the "haze" caused by gamma correction lifting near-black values, which otherwise results in washed-out colors (e.g., teal water instead of deep blue-violet) [S5, unofficial]. Use this when shadow depth and color accuracy in dark regions are critical [S5, unofficial].

### Chroma-Preserving Tonemapping
A post-processing step applied after tonemapping to restore saturation to bright colors desaturated by exponential tonemapping [S3, unofficial]. It mixes the original color with a luminance-only version using a factor of 1.32 [S3, unofficial]. Use this when vibrant highlights are being muted by the exposure compression [S3, unofficial].

### Final Fabric Enhancement
A postprocessing step that scales red and green channels by original intensities to create a fabric-like impression, avoiding the misunderstanding of rotating ribbons [S6, book]. It uses bump-mapped Phong intensity $\mathcal{B}_{x,y}$ and equations: $R(x,y) = \mathcal{B}_{x,y}(R(x,y)G(x,y) + R^2(x,y)) + e_{x,y} + \text{light}(\mathcal{L}_{x,y})$ and $G(x,y) = \mathcal{B}_{x,y}(R(x,y)G(x,y) + G^2(x,y)) + e_{x,y} + \text{light}(\mathcal{L}_{x,y})$ [S6, book]. Use this for visualizing tensor field data on surfaces where fabric-like texture is desired [S6, book].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Not given in the sources.
- **Strengths:** Simple to implement, provides artistic control over exposure and color, corrects for monitor non-linearity [S2][S3, unofficial].
- **Limitations:** Exponential tonemapping can desaturate bright colors and lift shadows, requiring additional steps (chroma preservation, black point correction) [S3, S5, unofficial].
- **When to Use:** Always as the final step in the rendering pipeline before display or file output [S3, S4, unofficial].
- **When Not to Use:** Not suitable for HDR display pipelines that preserve dynamic range without compression [S2].

## Background
Tonemapping and Gamma Correction originated from the need to display HDR content on LDR monitors [S2][S3, unofficial]. The exponential tonemapping operator $1 - \exp(-E \cdot I_F)$ is a common choice for its simplicity and perceptual uniformity [S2]. Gamma correction, based on the power law $C^{1/\gamma}$, was introduced to match the non-linear response of CRT monitors and remains standard for sRGB displays [S3, unofficial]. The specific variants like chroma preservation and black point correction were developed to address artifacts in specific rendering contexts, such as butterfly wing rendering and jellyfish visualization [S3, S5, unofficial].

## Sources
[S1] hexhowells/GoTracer, https://github.com/hexhowells/GoTracer, fetched 2026-10-26 (unofficial)
[S2] Real-time spectral scattering in large-scale natural participating media, https://doi.org/10.1145/1925059.1925074, fetched 2026-10-28
[S3] el-bakkali/manim-glsl-butterflies, https://github.com/el-bakkali/manim-glsl-butterflies, fetched 2026-10-26 (unofficial)
[S4] nihofm/volren: Simple OpenGL-based renderer for ..., https://github.com/nihofm/volren, fetched 2026-10-29 (unofficial)
[S5] el-bakkali/manim-glsl-jellyfish, https://github.com/el-bakkali/manim-glsl-jellyfish, fetched 2026-10-26 (unofficial)
[S6] Fabric-Like Visualization of Tensor Field Data on Arbitrary Surfaces in Image Space, https://doi.org/10.1007/978-3-642-27343-8_4, fetched 2026-10-27 (book)
