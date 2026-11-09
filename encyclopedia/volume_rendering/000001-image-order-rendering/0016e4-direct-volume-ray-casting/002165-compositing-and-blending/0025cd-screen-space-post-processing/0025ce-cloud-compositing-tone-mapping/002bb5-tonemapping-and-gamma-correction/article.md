# Tonemapping and Gamma Correction

Also called: Color Clamping, Exposure and Gamma Correction, Final image processing, Gamma Correction & PPM Output, PPM Writer, Tone Mapping, tonemap

## Overview
Tonemapping and Gamma Correction converts high-dynamic-range (HDR) radiance values into low-dynamic-range (LDR) pixel values suitable for display on standard monitors [S1, S3][S4, unofficial]. It solves the problem of HDR data exceeding the [0, 1] or [0, 255] range of output devices by compressing luminance while preserving perceptual contrast [S1]. Unlike the ACES Filmic Tone Mapping sibling, which uses a complex cinematic curve, this method typically employs a simple Reinhard operator combined with exposure scaling and gamma encoding [S1][S4, unofficial].

## How it works
The method processes the HDR fragment color $I_F$ (or $E$) through exposure adjustment, non-linear compression, and gamma encoding. Some implementations include additional post-processing steps for chroma preservation and black-point correction to address artifacts introduced by gamma lifting and exponential compression [S4, S6, unofficial].

### Inputs and Outputs
*   **Input:** HDR pixel color $C_{in}$ (typically `vec3` in [0, $\infty$)), exposure value $EV$ (scalar), and optionally samples per pixel $S$ [S1][S2, unofficial].
*   **Output:** LDR pixel color $C_{out}$ (typically `vec3` in [0, 1] or [0, 255]) [S1][S2, unofficial].

### Data Structures
*   **HDR Color:** Floating-point vector (e.g., `vec3` or `float3`). Format is often `GL_RGBA32F` in GPU implementations [S5, unofficial].
*   **Exposure Value:** Scalar float. Represents the exposure time $\Delta t = 2^{EV}$ [S1].

### Steps

1.  **Exposure Scaling:** Scale the HDR pixel value $E$ by $2^{EV}$ to simulate camera exposure [S1].
    $$ E' = 2^{EV} E $$
    *Note: Some implementations use a fixed exposure constant instead of $EV$. For example, [S4, unofficial] uses `col = 1 - exp(-col * 1.35)`.*

2.  **Tone Mapping (Reinhard):** Apply the Reinhard operator to compress the dynamic range [S1].
    $$ C_{tm} = \frac{E'}{E' + 1} $$
    *Alternative: [S3] uses $I_F = 1 - \exp(-E \cdot I_F)$.*

3.  **Gamma Correction:** Raise the tone-mapped result to the power of $1/\gamma$, where $\gamma = 2.2$ is standard for sRGB [S1][S4, unofficial].
    $$ C_{gamma} = C_{tm}^{\frac{1}{2.2}} $$
    *Note: Gamma lifting near-black values can cause visible haze in shadows [S6, unofficial].*

4.  **Optional: Black Point Correction:** Subtract channel-specific black points to restore deep shadows and correct color purity [S6, unofficial].
    $$ C_{bp} = \max(C_{gamma} - \mathbf{b}, \mathbf{0}) $$
    Where $\mathbf{b} = (0.0175, 0.0160, 0.0060)$ for Red, Green, Blue respectively [S6, unofficial].

5.  **Optional: Chroma Preservation:** Mix the original color with a luminance-only version to restore saturation lost during exponential tonemapping [S4, unofficial].
    $$ L = \text{dot}(C_{gamma}, (0.2126, 0.7152, 0.0722)) $$
    $$ C_{final} = \text{clamp}(\text{mix}((L, L, L), C_{gamma}, 1.32), 0.0, 1.0) $$

6.  **Optional: Explicit Tone Mapping (Learnable):** If using a differentiable module, apply white balance and a parameterized Camera Response Function (CRF) [S8].
    *   Apply white balance: $I_w = C_{in} \odot \theta_w$, where $\theta_w = [w_r, w_g, w_b]^T$ [S8].
    *   Apply CRF $g$: Interpolate using 256 control points in [0, 1] with leaky thresholding for saturation regions [S8].

### Pseudocode
The following pseudocode implements the standard Reinhard tone mapping with exposure and gamma correction, including optional black point and chroma preservation steps found in [S4, S6, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def tonemap_and_gamma(hdr_color, exposure_ev=0.0, apply_black_point=False, apply_chroma_preservation=False):
    """
    Converts HDR color to LDR using Reinhard tone mapping and gamma correction.
    
    Args:
        hdr_color: tuple or list of 3 floats (R, G, B) in HDR range [0, inf).
        exposure_ev: float, exposure value EV. Exposure time delta_t = 2^EV.
        apply_black_point: bool, whether to apply channel-wise black point correction.
        apply_chroma_preservation: bool, whether to apply chroma preservation mix.
        
    Returns:
        tuple of 3 floats (R, G, B) in range [0, 1].
    """
    # 1. Exposure Scaling
    # delta_t = 2^EV
    exposure_factor = math.pow(2, exposure_ev)
    r, g, b = hdr_color
    r_scaled = r * exposure_factor
    g_scaled = g * exposure_factor
    b_scaled = b * exposure_factor
    
    # 2. Reinhard Tone Mapping
    # C_tm = E' / (E' + 1)
    r_tm = r_scaled / (r_scaled + 1.0)
    g_tm = g_scaled / (g_scaled + 1.0)
    b_tm = b_scaled / (b_scaled + 1.0)
    
    # 3. Gamma Correction
    # C_gamma = C_tm ^ (1/2.2)
    gamma = 1.0 / 2.2
    r_gamma = math.pow(r_tm, gamma)
    g_gamma = math.pow(g_tm, gamma)
    b_gamma = math.pow(b_tm, gamma)
    
    # 4. Optional: Black Point Correction
    # Subtract vec3(0.0175, 0.0160, 0.0060) and clamp to 0
    if apply_black_point:
        r_bp = max(r_gamma - 0.0175, 0.0)
        g_bp = max(g_gamma - 0.0160, 0.0)
        b_bp = max(b_gamma - 0.0060, 0.0)
        r_gamma, g_gamma, b_gamma = r_bp, g_bp, b_bp
        
    # 5. Optional: Chroma Preservation
    # Mix with luminance version weighted by 1.32
    if apply_chroma_preservation:
        # Luminance weights from [S4, unofficial]
        lum = 0.2126 * r_gamma + 0.7152 * g_gamma + 0.0722 * b_gamma
        mix_factor = 1.32
        r_final = min(max(lum * (1.0 - mix_factor) + r_gamma * mix_factor, 0.0), 1.0)
        g_final = min(max(lum * (1.0 - mix_factor) + g_gamma * mix_factor, 0.0), 1.0)
        b_final = min(max(lum * (1.0 - mix_factor) + b_gamma * mix_factor, 0.0), 1.0)
        return (r_final, g_final, b_final)
    
    # 6. Clamp to [0, 1]
    r_final = min(max(r_gamma, 0.0), 1.0)
    g_final = min(max(g_gamma, 0.0), 1.0)
    b_final = min(max(b_gamma, 0.0), 1.0)
    
    return (r_final, g_final, b_final)
```

### Parameters
*   **Exposure Value ($EV$):** Scalar controlling exposure level. $\Delta t = 2^{EV}$ [S1]. Typical values depend on scene brightness; for synthetic scenes, values around 0-2 are common [S1].
*   **Gamma ($\gamma$):** Typically 2.2 for sRGB [S1][S4, unofficial].
*   **Black Point Vector:** $\mathbf{b} = (0.0175, 0.0160, 0.0060)$ for RGB [S6, unofficial].
*   **Chroma Mix Factor:** 1.32 [S4, unofficial].
*   **White Balance Parameters ($\theta_w$):** $[w_r, w_g, w_b]^T \in \mathbb{R}^3$, regularized to be positive [S8].
*   **CRF Control Points:** 256 points for piece-wise linear approximation [S8].

### Edge Cases and Pitfalls
*   **Gamma Lifting:** Raising near-zero values to $1/2.2$ lifts them significantly, causing visible haze in shadows [S6, unofficial]. Black point correction is required to mitigate this [S6, unofficial].
*   **Desaturation:** Exponential tonemapping desaturates bright colors, particularly the dominant channel [S4, unofficial]. Chroma preservation is needed to restore vividness [S4, unofficial].
*   **Clamping:** Values must be clamped to [0, 1] after tone mapping and gamma correction to prevent overflow in LDR displays [S2, S4, unofficial].
*   **Saturation in Training:** In differentiable rendering, saturation regions (values > 1 or < 0) can act as outliers. Leaky thresholding is used to propagate gradients through saturation regions [S8].

## Subtopics

### Channel-wise Black Point Correction
A post-processing step to correct color purity in shadows by subtracting a channel-specific black point vector (R: 0.0175, G: 0.0160, B: 0.0060) [S6, unofficial]. It addresses the issue where gamma lifting causes near-black values to appear as visible haze, resulting in washed-out colors (e.g., teal instead of deep blue-violet) [S6, unofficial].

### Chroma-Preserving Tonemapping
A post-processing step applied after tonemapping to restore saturation to bright colors desaturated by exponential compression [S4, unofficial]. It mixes the original color with a luminance-only version using a weight of 1.32 [S4, unofficial].

### Final Fabric Enhancement
A post-processing step for vector field visualization that scales Red and Green channels by original intensities to create a fabric-like impression [S7, book]. It uses bump-mapped Phong intensity $\mathcal{B}_{x,y}$ and equations: $R(x,y) = \mathcal{B}_{x,y}(R(x,y)G(x,y) + R^2(x,y)) + e_{x,y} + \text{light}(\mathcal{L}_{x,y})$ and similarly for Green [S7, book].

### Explicit Tone Mapping
A differentiable module converting HDR to LDR using white balance and a parameterized Camera Response Function (CRF) [S8]. It uses 256 control points for the CRF and leaky thresholding for saturation regions to enable gradient propagation during training [S8].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per pixel for standard Reinhard tone mapping [S1]. The explicit tone mapping with 256 control points involves interpolation, which is also effectively $O(1)$ per pixel but with a higher constant factor [S8].
*   **Memory Complexity:** $O(1)$ per pixel for standard methods. Explicit tone mapping requires storing 256 control points per channel if learned [S8].
*   **Performance:** The standard Reinhard method is extremely fast and suitable for real-time rendering [S1]. GPU implementations use compute shaders dispatched at resolution size [S5, unofficial].
*   **Strengths:** Simple, fast, and effective for converting HDR to LDR [S1]. Exposure control allows for flexible scene brightness adjustment [S1].
*   **Limitations:** Can cause shadow haze without black point correction [S6, unofficial]. Can desaturate bright colors without chroma preservation [S4, unofficial]. The Reinhard operator may not preserve perceptual contrast as well as more complex curves like ACES [S1].
*   **When to use:** When a simple, fast tone mapping is sufficient for visualization or evaluation [S1]. When working with HDR-NeRF or similar radiance fields [S1, S8].
*   **When not to use:** When high-fidelity cinematic color grading is required (use ACES Filmic instead) [S1]. When shadow detail is critical and gamma lifting artifacts are unacceptable without additional processing [S6, unofficial].

## Background
The method originates from classical tone mapping techniques, specifically the Reinhard operator [S1][S4, unofficial]. It is widely used in neural radiance field (NeRF) research to generate LDR ground truth views from HDR renders for evaluation [S1]. The explicit tone mapping variant was introduced in HDR-Plenoxels to enable differentiable tone mapping for radiance field optimization [S8]. The chroma preservation and black point correction steps were developed in the context of Manim-based visualizations to correct artifacts from gamma lifting and exponential compression [S4, S6, unofficial].

## Sources
[S1] HDR-NeRF: High Dynamic Range Neural Radiance Fields, https://xhuangcv.github.io/hdr-nerf/, fetched 2026-10-27
[S2] hexhowells/GoTracer, https://github.com/hexhowells/GoTracer, fetched 2026-10-26 (unofficial)
[S3] Real-time spectral scattering in large-scale natural participating media, https://doi.org/10.1145/1925059.1925074, fetched 2026-10-28
[S4] el-bakkali/manim-glsl-butterflies, https://github.com/el-bakkali/manim-glsl-butterflies, fetched 2026-10-26 (unofficial)
[S5] nihofm/volren: Simple OpenGL-based renderer for ..., https://github.com/nihofm/volren, fetched 2026-10-29 (unofficial)
[S6] el-bakkali/manim-glsl-jellyfish, https://github.com/el-bakkali/manim-glsl-jellyfish, fetched 2026-10-26 (unofficial)
[S7] Fabric-Like Visualization of Tensor Field Data on Arbitrary Surfaces in Image Space, https://doi.org/10.1007/978-3-642-27343-8_4, fetched 2026-10-27 (book)
[S8] HDR-Plenoxels: Self-Calibrating High Dynamic Range Radiance Fields, https://doi.org/10.1007/978-3-031-19824-3_23, fetched 2026-10-26
