# Hardware Shading via Pixel Texture

Also called: Quantized Normal Shading

## Overview
This method adds shading to texture-memory based volume rendering to increase visual quality with negligible cost, addressing the common lack of shading in previous hardware-accelerated approaches [S1]. It encodes surface normals into a 2D pixel texture using a 16-bit quantization scheme and nearest-neighbor interpolation to avoid hardware interpolation redundancy and truncation [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume data $V(\mathbf{x})$, gradient-based normals $\mathbf{n}(\mathbf{x})$, and a lighting model.
*   **Output:** Shaded color $C_{shaded}$ composited into the final image.

**Data Structures:**
*   **Shading Pixel Texture:** A 2D texture where each texel stores a quantized normal.
    *   **Layout:** 16-bit encoding.
    *   **Channel Mapping:** The first 8 bits of the quantized normal are stored in the Red channel; the second 8 bits are stored in the Green channel [S1, S2].
    *   **Interpolation:** Nearest-neighbor interpolation is used for lookups to prevent redundancy and truncation errors inherent in hardware linear interpolation of quantized data [S1, S2].

**Steps:**
1.  **Normal Quantization:** Compute surface normals from the volume gradient. Encode each normal component into a 16-bit quantized value [S1, S2].
2.  **Texture Generation:** Generate the shading pixel texture on a per-view basis in software [S1, S2].
    *   *Alternative Hardware Method:* Render quantized normals as points with hardware lighting and copy the result from the frame buffer to the pixel texture [S2].
3.  **Lighting Lookup:** Use a 2D pixel texture to light the quantized normals. This is equivalent to a 16-bit 1D nearest-neighbor pixel texture lookup [S1, S2].
4.  **Rendering Pipeline (SGI Octane2/V-Series):**
    *   Render the current slice to a scratch/auxiliary buffer [S2].
    *   If shading is enabled, render the matching slice from the shading volume and modulate (multiply) it with the current slice [S2].
    *   Use a `glCopyPixels()` operation to pixel texture the scratch buffer content, as pixel texture does not occur directly on a per-fragment basis during rasterization on this hardware [S2].
    *   Copy the slice from the scratch buffer and blend it with previously rendered slices in the frame buffer [S2].
5.  **Rendering Pipeline (NVIDIA GeForce3):**
    *   Utilize four-way multitexturing. The textures include the volume data (VGH) and pre-computed RGBA shade volumes [S1].

**Parameters:**
*   **Quantization:** 16-bit encoding for normals [S2].
*   **Threshold:** Not specified for this specific shading step, but follows the parent's early ray termination threshold (typically 1.0 or 0.95) [S9, S12, S27, unofficial].

**Edge Cases and Pitfalls:**
*   **Hardware Flexibility:** Some hardware implementations are not flexible enough to support rendering quantized normals as points with hardware lighting and copying to a pixel texture [S2].
*   **Interpolation Errors:** Using standard hardware interpolation (linear) on quantized normals causes redundancy and truncation; nearest-neighbor is required [S1, S2].

## Subtopics

### Quantized Normal Shading via Pixel Texture
A shading method that encodes surface normals into a 2D pixel texture to avoid hardware interpolation redundancy and truncation, using nearest-neighbor interpolation [S2]. It encodes the first eight bits of the quantized normal into the red channel and the second eight into the green channel [S2]. The shading pixel texture is generated on a per-view basis in software, or alternatively by rendering quantized normals as points with hardware lighting and copying from the frame buffer [S2]. On hardware like the SGI Octane2, where pixel texture does not occur directly on a per-fragment basis during rasterization, the slice is rendered to a scratch/auxiliary buffer, pixel textured via `glCopyPixels()`, and then blended with previously rendered slices in the frame buffer [S2]. A key optimization is that the pixel texture operation is only required on the portion of the volume currently changing (e.g., when the viewpoint moves, not when the transfer function is manipulated) [S2]. A pitfall is that some hardware implementations lack the flexibility to support the point-rendering-to-texture copy operation [S2].

### Line Bundles
A technique relying on graphics hardware to create texture by rendering and compositing many semitransparent line segments to produce an anisotropic volume rendering result [S3, unofficial]. For each data point, it renders pre-computed sets of line segments using a splatting analogy, slightly perturbing their colors to provide a colored pattern [S3, unofficial]. These line bundles are composited together in back-to-front order [S3, unofficial]. This allows viewing velocity magnitude like normal volume rendering while clearly evidencing directionality, which standard volume rendering may lack [S3, unofficial].

## Operating characteristics
*   **Performance Cost:** The performance cost of generating the shading pixel texture in software is minimal [S1].
*   **Quality:** Dramatically increases the quality of renderings by expressing shape information through shading [S1].
*   **Limitations:** Dependent on hardware support for pixel textures and specific operations like `glCopyPixels()` or point rendering to texture [S2].

## Background
This method was developed to address the lack of shading in texture-memory based volume rendering [S1]. It was implemented on the SGI Octane 2 with V-series graphics cards and the NVIDIA GeForce3 series [S2]. The approach leverages the observation that when the transfer function is manipulated, the viewpoint is static, and vice versa, allowing for efficient updates of the shading texture [S2].

## Sources
[S1] PDF Interactive Volume Rendering Using Multi-Dimensional Transfer Functions ..., https://publications.sci.utah.edu/publications/jmk01/simian-paper.pdf, fetched 2026-10-27
[S2] PDF Interactive Volume Rendering Using Multi-Dimensional Transfer Functions ..., https://collections.lib.utah.edu/dl_files/d4/e4/d4e47bfc09e54eee9f532e4f22b17e9214f4c755.pdf, fetched 2026-10-27
[S3] Flow Visualization Techniques for CDF using Volume Rendering, https://www.osti.gov/servlets/purl/790542, fetched 2026-10-26 (unofficial)
