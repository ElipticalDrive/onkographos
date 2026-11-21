# Multi-Texture Correct Blending


## Overview
Multi-Texture Correct Blending is a technique that uses register combiners to combine $n$ texture images on a single polygon such that the resulting fragment blends into the frame buffer with the exact same result as rendering $n$ separate polygons with single textures [S1]. It solves the problem of incorrect alpha blending when multiple volume slices are rendered simultaneously on a single geometry, ensuring mathematical equivalence to the standard back-to-front polygon rendering order [S1]. It sets itself apart from standard multi-texturing by explicitly computing the composite alpha term $(1 - A_{t0})(1 - A_{t1})$ to preserve the correct transparency accumulation, whereas standard multi-texturing often fails to account for the multiplicative nature of alpha subtraction in sequential blending [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Two texture images (Texture 0 and Texture 1), each providing RGB color values ($C_{t0}, C_{t1}$) and Alpha values ($A_{t0}, A_{t1}$). The current frame buffer color ($C_d$).
*   **Outputs:** A single fragment with final color $C_s$ and final alpha $A_s$, which is then blended into the frame buffer.

**Data Structures and Layout:**
The method relies on the OpenGL Register Combiner architecture (specifically NV_register_combiner or similar extensions). It utilizes two combiner stages:
1.  **General Combiner 0:**
    *   **RGB Portion:** Calculates the intermediate terms $C_{t0} A_{t0}$ and $C_{t1} A_{t1}$.
    *   **Alpha Portion:** Calculates the term $(1 - A_{t0})(1 - A_{t1})$.
2.  **Final Combiner:**
    *   Takes the RGB output from Combiner 0 and the Alpha output from Combiner 0.
    *   Computes the final color $C_s$ and passes the alpha $A_s$ directly.

**Numbered Steps:**
1.  **Setup Blending Function:** Configure the OpenGL blending function to `glBlendFunc(GL_ONE, GL_SRC_ALPHA)`. This ensures the fragment color is added to the destination buffer weighted by its alpha, and the destination buffer is weighted by $(1 - A_s)$ [S1].
2.  **Configure Combiner 0 RGB:** Program the RGB portion of General Combiner 0 to compute the products of texture color and alpha for both textures. Specifically, it calculates $C_{t0} A_{t0}$ and $C_{t1} A_{t1}$ [S1].
3.  **Configure Combiner 0 Alpha:** Program the Alpha portion of General Combiner 0 to compute the product of the inverted alphas: $(1 - A_{t0})(1 - A_{t1})$ [S1].
4.  **Configure Final Combiner RGB:** Route the RGB outputs from Combiner 0 to the Final Combiner. Program it to calculate the final color $C_s$ using the formula:
    $$C_s = C_{t0} A_{t0} (1 - A_{t1}) + C_{t1} A_{t1}$$
    This effectively combines the contribution of the first texture (modulated by the transparency of the second) and the second texture [S1].
5.  **Configure Final Combiner Alpha:** Route the Alpha output from Combiner 0 directly to the output register of the Final Combiner, setting the fragment alpha to:
    $$A_s = (1 - A_{t0}) (1 - A_{t1})$$
6.  **Render:** Draw the single polygon with both textures bound. The hardware executes the combiner operations, producing the fragment $(C_s, A_s)$.
7.  **Blend:** The fragment is blended into the frame buffer using the setup blending function:
    $$C_{d00} = C_s \cdot 1 + C_d \cdot A_s$$
    Substituting the values:
    $$C_{d00} = C_{t1} A_{t1} + C_{t0} A_{t0} (1 - A_{t1}) + C_d (1 - A_{t0}) (1 - A_{t1})$$
    This matches the result of rendering two separate polygons back-to-front [S1].

**Formulas:**
*   **Standard Sequential Blending (Reference):**
    Blending first polygon: $C_{d0} = C_{t0} A_{t0} + C_d (1 - A_{t0})$ [S1].
    Blending second polygon: $C_{d00} = C_{t1} A_{t1} + C_{d0} (1 - A_{t1})$ [S1].
    Expanded: $C_{d00} = C_{t1} A_{t1} + C_{t0} A_{t0} (1 - A_{t1}) + C_d (1 - A_{t0}) (1 - A_{t1})$ [S1].
*   **Multi-Texture Correct Blending Output:**
    Final Color: $C_s = C_{t0} A_{t0} (1 - A_{t1}) + C_{t1} A_{t1}$ [S1].
    Final Alpha: $A_s = (1 - A_{t0}) (1 - A_{t1})$ [S1].

**Parameters:**
*   **Blending Function:** `GL_ONE` for source, `GL_SRC_ALPHA` for destination [S1].
*   **Texture Environment:** `GL_REPLACE` is assumed for the base texture sampling before combiner processing [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The calculation involves multiple multiplications and additions. Fixed-point hardware or limited precision in register combiners may introduce artifacts, particularly when alpha values are very small or very close to 1 [G].
*   **Number of Textures:** The provided source details the setup for two textures ($n=2$). Extending this to $n > 2$ requires additional combiner stages to compute the product $(1 - A_{t0})(1 - A_{t1})...(1 - A_{t(n-1)})$ and the weighted sum of colors. The source does not provide the specific layout for $n > 2$ [S1].
*   **Hardware Support:** This method requires support for the Register Combiner extension (e.g., `GL_NV_register_combiners`). It is not available on modern programmable shader pipelines without emulation or equivalent logic in fragment shaders [G].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** Constant time per fragment for the combiner operations, independent of the number of slices if $n$ is fixed and small. However, the complexity of the combiner setup grows with $n$.
*   **Memory:** Requires storage for the combiner state and the texture images. No additional frame buffer memory is used compared to standard rendering.

**Measured Performance:**
*   The source does not provide specific performance metrics (FPS, memory usage) for this specific technique [S1].

**Scaling:**
*   Scaling to more than two textures requires additional combiner stages. The source only describes the two-texture case [S1].

**Strengths and Limitations:**
*   **Strengths:** Provides mathematically exact equivalence to sequential polygon rendering, avoiding the visual artifacts of standard multi-texturing alpha blending [S1].
*   **Limitations:** Limited to the number of available combiner stages and texture units. Requires specific hardware extensions (Register Combiners) that are obsolete in modern graphics APIs [G].

**When to Use:**
*   Use when exact alpha blending equivalence is required and the hardware supports Register Combiners. Not suitable for modern GPU architectures without shader-based equivalents [G].

## Background
**Origin:**
The technique was introduced in the context of interactive volume rendering on standard PC graphics hardware, specifically to leverage rasterization hardware for efficient volume visualization [S1].

**History and Development:**
It was presented in the paper "Interactive Volume Rendering on Standard PC Graphics Hardware" [S1]. The method exploits the capabilities of the OpenGL Register Combiner extension to perform complex blending operations in hardware, avoiding the need for multiple draw calls and depth sorting for every slice [S1].

**Where it is used:**
It is used in volume rendering applications that require high-quality alpha blending on older hardware supporting the Register Combiner extension [S1].

## Sources
[S1] Interactive Volume Rendering on Standard PC Graphics Hardware, https://courses.cs.duke.edu/spring03/cps296.8/papers/VolVisforPCHardware.pdf, fetched 2026-10-28
