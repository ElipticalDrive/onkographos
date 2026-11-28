# Hardware Shading via Pixel Texture

Also called: Quantized Normal Shading, Volume Ray Tracing via 2D Texture Packing

## Overview
Hardware Shading via Pixel Texture is a technique for adding lighting calculations to texture-memory based volume rendering, addressing the common limitation where such methods lack shading [S2]. It distinguishes itself from standard hardware interpolation by encoding surface normals into a 2D pixel texture and using nearest-neighbor lookups to avoid redundancy and truncation errors [S1, S2]. This approach increases rendering quality with negligible cost by decoupling shading from the primary volume sampling pipeline [S2].

## How it works
**Inputs:**
*   Volume data (density and radiance fields).
*   View direction.
*   Quantized normal data (pre-computed or generated per-view).

**Outputs:**
*   Shaded volume image.

**Data Structures:**
*   **2D Pixel Texture (Shading Texture):** Stores lighting information. Dimensions match the output resolution.
*   **Quantized Normal Encoding:**
    *   Format: 16-bit integer per normal vector.
    *   Channel Layout: The first 8 bits of the quantized normal are stored in the Red channel; the second 8 bits are stored in the Green channel [S2].
    *   Interpolation Mode: Nearest-neighbor [S2].

**Steps:**
1.  **Volume Ray Tracing/Sampling:**
    *   Pack all sample points into a 2D texture representation [S1].
    *   Draw a quad of the same size as the texture to trigger the pixel shader [S1].
    *   Perform volume ray tracing within the shader [S1].
    *   Terminate ray tracing if the ray exits the volume [S1].
    *   *Note:* If using ray marching with RBFs, intersect RBFs with slices perpendicular to the view direction, render slices far-to-near with alpha blending (`GL_ONE`, `GL_SRC_ALPHA`), and retrieve density $D(x)$ and source radiance $L_x$ from 3D textures [S1]. Set RGB output to $D(x)L_x$ and alpha to differential transmittance [S1].

2.  **Normal Encoding:**
    *   Encode surface normals using a 16-bit quantization scheme [S2].
    *   Split the 16-bit value: assign the first 8 bits to the Red channel and the second 8 bits to the Green channel of the pixel texture [S2].

3.  **Shading Lookup:**
    *   Perform a nearest-neighbor lookup on the 2D pixel texture containing the quantized normals [S2].
    *   Use nearest-neighbor interpolation specifically to avoid redundancy and truncation errors associated with standard hardware interpolation [S1, S2].

4.  **Lighting Calculation:**
    *   Light the quantized normals using the 2D pixel texture [S2].
    *   The process is equivalent to a 16-bit 1D nearest-neighbor pixel texture lookup or an 8-bit per-axis 2D nearest-neighbor texture lookup [S2].
    *   Generate the shading pixel texture on a per-view basis in software [S2].

**Formulas:**
*   **Radiance Retrieval (if using RBF ray marching):**
    $$ \text{RGB} = D(x) L_x $$
    Where $D(x)$ is density and $L_x$ is source radiance retrieved from 3D textures [S1].
*   **Normal Encoding:**
    $$ \text{Red} = \text{QuantizedNormal} \gg 8 $$
    $$ \text{Green} = \text{QuantizedNormal} \ \& \ 0xFF $$
    Where `>>` is bitwise right shift and `&` is bitwise AND [S2].

**Parameters:**
*   **Quantization Bits:** 16 bits per normal [S2].
*   **Interpolation:** Nearest-neighbor [S2].

**Edge Cases and Pitfalls:**
*   **Interpolation Artifacts:** Using linear interpolation for normals causes redundancy and truncation errors; nearest-neighbor is required [S1, S2].
*   **Performance Cost:** Generating the shading pixel texture per-view in software has minimal cost but could be performed in hardware [S2].

## Subtopics

### [Quantized Normal Shading via Pixel Texture](6549-quantized-normal-shading-via-pixel-texture.md)
A variant that explicitly encodes surface normals into a 2D pixel texture to provide shading while avoiding redundancy and truncation errors. It uses a dedicated 2D texture lookup for lighting calculations. Choose this when high-quality shading is needed without interpolation artifacts.

### [Line Bundles](7701-line-bundles.md)
A technique for visualizing 3D vector fields by compositing semitransparent line segments. It builds on this method by extending the concept of splats to composite collections of antialiased lines in a back-to-front manner. Choose this for directional vector field visualization.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** The performance cost of generating the shading pixel texture is minimal [S2].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Dramatically increases rendering quality with negligible cost; avoids interpolation artifacts for normals [S2].
*   **Limitations:** Previous work often lacked shading; requires per-view generation of shading texture (though minimal cost) [S2].
*   **When to use:** When texture-memory based volume rendering needs shading without significant performance overhead.
*   **When not to use:** When hardware interpolation of normals is preferred despite potential artifacts, or when per-view shading texture generation is prohibitive.

## Background
Originating from work on interactive volume rendering using multi-dimensional transfer functions, this method was introduced to address the lack of shading in texture-memory based volume rendering [S2]. It was developed to provide a natural and efficient way to express information about the shape of structures in the volume [S2]. The technique was implemented on hardware such as the Octane2 and GeForce3, utilizing multi-texturing pipelines [S2].

## Sources
[S1] Gradient‐based Interpolation and Sampling for Real‐time Rendering of Inhomogeneous, Single‐scattering Media, https://doi.org/10.1111/j.1467-8659.2008.01343.x, fetched 2026-10-27
[S2] PDF Interactive Volume Rendering Using Multi-Dimensional Transfer Functions ..., https://publications.sci.utah.edu/publications/jmk01/simian-paper.pdf, fetched 2026-10-27
