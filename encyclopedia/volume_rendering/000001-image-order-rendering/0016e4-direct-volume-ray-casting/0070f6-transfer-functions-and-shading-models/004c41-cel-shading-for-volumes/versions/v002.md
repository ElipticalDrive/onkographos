# Cel Shading for Volumes

Also called: Toon Shading for Volume Rendering, Toon Shading for Volumes, XToon, XToon Volume Rendering

## Overview
Cel Shading for Volumes (XToon) is a direct volume ray casting technique that applies stylized "toon" shading effects—including tone detail, specular highlights, and backlighting—to volumetric data [S1, unofficial]. It differs from standard direct volume rendering by replacing continuous Phong illumination with discrete, transfer-function-driven shading components to produce sharp, comic-book-like contours and highlights, while retaining the semi-transparent accumulation of the parent method [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume texture $V$, ray start/stop positions per pixel, camera position, light position, and shading parameters.
*   **Output:** A 2D color buffer with alpha compositing.

**Data Structures:**
*   **Volume Texture:** 3D texture, typically `GL_LUMINANCE` format with `GL_UNSIGNED_BYTE` data [S1, unofficial].
*   **Ray Textures:** Two 2D textures (`RayStart`, `RayStop`) storing the 3D start and stop coordinates for each pixel's ray [S1, unofficial].
*   **Transfer Function Textures:** Four 2D textures (`TransferFunc_01` to `TransferFunc_04`) used for XToon lookups [S1, unofficial].
*   **Accumulators:** Per-ray accumulators for gradient magnitude (`gradient_acc`) and transfer function values (`trans_acc`) [S1, unofficial].

**Parameters:**
*   `StepLength`: Step size along the ray. Typical value: 0.01 [S1, unofficial].
*   `Threshold`: Minimum volume value to trigger gradient computation [S1, unofficial].
*   `NSamples`: Number of refinement steps for hitpoint [S1, unofficial].
*   `Jitter`: Integer flag (0 or 1) to enable stochastic jittering [S1, unofficial].
*   `NoiseDelta`: Float parameter for jitter scale [S1, unofficial].
*   `GradientScale`: Float multiplier for gradient magnitude before transfer function lookup [S1, unofficial].
*   `GradientDelta`: Float multiplier for gradient vector [S1, unofficial].
*   `TransScale`: Float multiplier for volume density value [S1, unofficial].
*   `Alpha`: Float parameter controlling alpha mixing [S1, unofficial].
*   `r`: Parameter for tone detail range scaling [S1, unofficial].
*   `s`: Specular shininess exponent [S1, unofficial].
*   `BacklightDetail`: Backlight exponent [S1, unofficial].
*   `xtoonFlag`: Enable flag for XToon (1 = enabled) [S1, unofficial].
*   `xtoonTypeTone`, `xtoonTypeSpec`, `xtoonTypeBack`: Enable flags for specific XToon components [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:** For each pixel, compute screen coordinates `coord = (vPosition.xy / vPosition.w + 1.0) / 2.0`. Sample `rayStart` and `rayStop` from their respective 2D textures. If `rayStart == rayStop`, discard the pixel [S1, unofficial].
2.  **Ray Setup:** Calculate `ray = rayStop - rayStart`, `rayLength = length(ray)`, and `stepVector = StepLength * ray / rayLength`. Initialize `pos = rayStart`, `dst = vec4(0)`, `gradient_acc = vec4(0)`, `trans_acc = vec4(0)` [S1, unofficial].
3.  **Jittering:** If `Jitter == 1`, offset `rayStart` by `stepVector * rand(vPosition.xy / NoiseDelta)`, where `rand` is a pseudo-random hash function [S1, unofficial].
4.  **Ray Casting Loop:** Iterate while `dst.a < 1.0` and `rayLength > 0.0`:
    a.  **Sample Volume:** Compute scalar value $V = \text{dot}(\mathbf{V}_{tex}, \mathbf{V}_{tex})$ where $\mathbf{V}_{tex} = \text{texture3D}(Volume, pos).xyz$ [S1, unofficial].
    b.  **Threshold Check:** If $V \le Threshold$, reset accumulators if they exceed `Threshold` (see step 4e), advance ray (step 4e), and continue [S1, unofficial].
    c.  **Gradient Computation:** If $V > Threshold$, compute Sobel-like gradient components $g_x, g_y, g_z$ using 3D texture lookups at offsets defined by $e = \text{vec4}(\text{vec3}(-1.0, 1.0, 0.0)/256.0, 0.0)$ [S1, unofficial].
        *   $g_x = 2.0 \cdot |V(pos+e.yww) - V(pos+e.xww)| + \dots$ (sum of absolute differences as per source [S1, unofficial]).
        *   $g_y, g_z$ computed similarly with respective offsets [S1, unofficial].
        *   Compute `sobelResult = mask.xwww * g_x + mask.wxww * g_y + mask.wwxw * g_z + mask.wwwx` where `mask = vec4(1.0, 0.0, 0.0, 0.0)` [S1, unofficial].
        *   Accumulate: `gradient_acc += abs(sobelResult)` [S1, unofficial].
    d.  **Hitpoint Refinement:** To reduce slicing artifacts, refine the position:
        *   Initialize `s = -stepVector * 0.5`.
        *   Update `pos += s` and re-sample $V$.
        *   Loop `NSamples` times: If $V > Threshold$, `s *= 0.5`, `pos += s`; else `s *= -0.5`, `pos += s`. Re-sample $V$ each iteration [S1, unofficial].
    e.  **Shading and Compositing:** If refined $V > Threshold$:
        *   Compute gradient vector `vGradient = gradient_acc.rgb * GradientDelta` and normalize to `nGradient`. Compute `GradientMag = length(vGradient)` [S1, unofficial].
        *   Compute lighting vectors: `viewDir = -normalize(eyePosition - pos)`, `normal = normalize(gl_NormalMatrix * nGradient)`, `NdotL = dot(normal, LightPosition)`, `NdotV = dot(normal, viewDir)` [S1, unofficial].
        *   Compute reflection `reflection = normalize(((2.0 * normal) * NdotL) - LightPosition)` and `VdotR = max(0.0, dot(viewDir, reflection))` [S1, unofficial].
        *   **XToon Shading (if `xtoonFlag == 1`):**
            *   **Tone Detail:** If `xtoonTypeTone == 1`, compute tone value $D$ using `xtoonToneDetail(V * TransScale, texture3D(Volume, pos).z)` and lookup `trans_val = lookupTransferTex(GradientMag * GradientScale, D)` [S1, unofficial]. Accumulate `trans_acc += trans_val` [S1, unofficial].
            *   **Specular:** If `xtoonTypeSpec == 1`, compute specular value $D = VdotR^s$ and lookup `spec_val = lookupTransferTex(NdotL, D)` [S1, unofficial].
            *   **Backlight:** If `xtoonTypeBack == 1`, compute backlight value $D = NdotV^{BacklightDetail}$ and lookup `backlight_val = lookupTransferTex(NdotL, D)` [S1, unofficial].
            *   **Mixing:** `color_sample = vec4(mix(mix(mix(backlight_val.rgb, trans_val.rgb, 0.5), spec_val.rgb, 0.5), trans_acc.rgb, 0.5), mix(trans_acc.a, gradient_acc.a, Alpha))` [S1, unofficial].
        *   **Standard Shading (if `xtoonFlag != 1`):** `trans_val = lookupTransferTex(NdotL, V * TransScale)`, `color_sample = trans_val` [S1, unofficial].
        *   **Blend:** `color_sample.rgb *= color_sample.a`, `dst = (1.0 - dst.a) * color_sample + dst`. Break loop [S1, unofficial].
    f.  **Reset Accumulators:** If `length(gradient_acc) > Threshold`, reset `gradient_acc = vec4(0)`. If `length(trans_acc) > Threshold`, reset `trans_acc = vec4(0)` [S1, unofficial].
    g.  **Advance Ray:** `pos += stepVector`, `rayLength -= StepLength` [S1, unofficial].
5.  **Output:** Write `dst` to `gl_FragColor` [S1, unofficial].

**Formulas:**
*   **Tone Detail ($D$):**
    $$ D = 1.0 - \frac{\log(density / densityMin)}{\log(densityMax - densityMin)} $$
    Adjustments apply if $density < densityC - densityMin$ or $density > densityC + densityMin$, where $densityMin = 0.01$, $densityMax = densityMin \cdot r$, and $densityC$ is the current density [S1, unofficial].
*   **Specular Intensity ($D$):**
    $$ D = VR^s $$
    Where $VR$ is the view-reflection dot product and $s$ is the shine factor [S1, unofficial].
*   **Backlight Intensity ($D$):**
    $$ D = NV^{BacklightDetail} $$
    Where $NV$ is the normal-view dot product [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Slicing Artifacts:** Mitigated by the hitpoint refinement loop, but insufficient `NSamples` may leave artifacts [S1, unofficial].
*   **Gradient Noise:** The Sobel-like operator is sensitive to noise in the volume data; smoothing may be required [S1, unofficial].
*   **Boundary Conditions:** Texture wrapping is `GL_CLAMP`; ensure ray start/stop are within bounds [S1, unofficial].
*   **Accumulator Reset:** The reset condition `length(...) > Threshold` may cause premature clearing if `Threshold` is set too low [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples per ray and the number of rays (pixels). The gradient computation and refinement loop add constant overhead per sample.
*   **Memory Complexity:** Dominated by the 3D volume texture and 2D ray/transfer function textures. Accumulators are per-ray, stored in registers/locals.
*   **Performance:** Dependent on `StepLength` and `NSamples`. Stochastic jittering adds minimal overhead. The use of 2D transfer function lookups for shading is efficient compared to complex lighting calculations.
*   **Strengths:** Produces stylized, high-contrast images with clear internal structures. Efficient implementation using fragment shaders.
*   **Limitations:** Stylized appearance may not be suitable for scientific visualization requiring photorealism. Sensitive to volume data quality (noise).
*   **When to Use:** Artistic visualization, educational materials, or when specific stylistic effects (contours, highlights) are desired.
*   **When Not to Use:** When physical accuracy or subtle density variations are critical.

## Background
The method is implemented in the "XToonVolume" project by Andre Rogers [S1, unofficial]. It extends direct volume ray casting with specific XToon shading techniques, including tone detail, specular highlights, and backlighting, using gradient-based normal estimation [S1, unofficial].

## Sources
[S1] andrerogers/XToonVolume, https://github.com/andrerogers/XToonVolume, fetched 2026-10-26 (unofficial)
