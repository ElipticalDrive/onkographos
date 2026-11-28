# Cabral Texture Mapping

Also called: Filtered Back Projection via Texture Mapping, Texture Mapping Backprojection, Trilinear interpolation of texture mapped slices

## Overview
Cabral Texture Mapping is a volume rendering method that implements the filtered back projection (FBP) algorithm for Computed Tomography (CT) using hardware-accelerated texture mapping and an accumulation buffer [S2, unofficial]. It differs from standard ray-casting or standard texture-slicing approaches by treating the reconstruction process as a series of texture-mapped slices that are blended back into the image space, effectively performing the back-projection step via rasterization [S2, unofficial]. While it produces unshaded images, it suffers from the drawback that the volume must be re-shaded and re-loaded whenever viewing parameters change, and non-linear transfer functions are not interpolated correctly by the texture hardware [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume dataset (CT scan data).
*   **Output:** A 2D rendered image (unshaded).

**Data Structures:**
*   **Volume Data:** 3D grid of scalar values.
*   **Texture Map:** The volume data is loaded into 3D texture memory.
*   **Accumulation Buffer:** A hardware buffer used to accumulate the contributions of the back-projected slices.

**Steps:**
1.  **Texture Loading:** Load the 3D volume data into the graphics hardware's 3D texture memory [S2, unofficial].
2.  **Slice Generation:** Generate a series of parallel slicing planes that intersect the volume. These planes represent the back-projection paths [S1, unofficial].
3.  **Trilinear Interpolation:** For each pixel in the slicing plane, trilinearly interpolate the scalar value from the 3D texture map based on the plane's position relative to the volume [S1, unofficial].
4.  **Back-to-Front Blending:** Render the texture-mapped slices into the accumulation buffer in back-to-front order relative to the viewer. The hardware blends the slices into the frame buffer [S1, unofficial].
5.  **Accumulation:** The accumulation buffer sums the contributions of all slices, effectively performing the filtered back projection integral [S2, unofficial].

**Formulas:**
The blending equation used during the accumulation step is the standard alpha blending equation:
$$ C_{dest} = C_{src} \cdot A_{src} + C_{dest} \cdot (1 - A_{src}) $$
where $C_{src}$ and $A_{src}$ are the color and opacity derived from the trilinearly interpolated scalar value, and $C_{dest}$ is the current value in the accumulation buffer [S1, unofficial][S3, unofficial].

**Parameters:**
*   **Number of Slices:** Determined by the desired resolution and viewing angle. Not given in the sources.
*   **Texture Filtering:** Trilinear interpolation is specified [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Transfer Functions:** Non-linear transfer functions are not interpolated correctly by the texture hardware, leading to potential inaccuracies in shading [S2, unofficial].
*   **Viewing Parameters:** The volume must be re-shaded and re-loaded every time viewing parameters change, which can be computationally expensive [S2, unofficial].

## Subtopics

### Multi-resolution Representation for Intermediate Template
A technique to overcome resolution limitations and glancing angle projection artifacts by generating the intermediate template at different resolutions for different areas, rather than using a single finest resolution for the whole template [S3, unofficial]. It defines resolution intervals $\Delta u = 1/(x_d - 1)$ and $\Delta v = 1/(y_d - 1)$. Continuous variables $u$ and $v$ are used to adjust the origin and direction of the tracing ray during Direct Volume Rendering (DVR) or Direct Surface Rendering (DSR) generation of the intermediate template, allowing different resolution intervals to be used [S3, unofficial]. This is useful because different areas may have different resolution requirements (e.g., wrinkles around eyes), making rendering the whole template at the finest resolution impractical and unnecessary [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Requires 3D texture memory for the volume and an accumulation buffer. Not given in the sources.
*   **Measured Performance:** Implemented on four Raster Manager Reality Engine Onyx machines [S2, unofficial].
*   **Strengths:** Leverages hardware acceleration for back-projection; suitable for real-time CT visualization on specialized hardware [S2, unofficial].
*   **Limitations:** Unshaded images; inefficient update when viewing parameters change; poor support for non-linear transfer functions [S2, unofficial].
*   **When to use:** When using specialized hardware (like Onyx) and requiring fast back-projection for CT data.
*   **When not to use:** When non-linear transfer functions or dynamic viewing parameter updates are critical.

## Background
Cabral Texture Mapping was introduced by Cabral et al. [S2, unofficial]. It was implemented on four Raster Manager Reality Engine Onyx machines to implement the filtered back projection CT algorithm [S2, unofficial]. The method combines texture mapping with an accumulation buffer to achieve real-time volume rendering for medical imaging applications [S2, unofficial].

## Sources
[S1] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
[S2] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S3] Texture Mapping Volume Objects, https://doi.org/10.2312/vvg.20051006, fetched 2026-10-30 (unofficial)
