# Multi-pass multi-view PBR

Also called: 8 MV, Conventional Multi-View Rendering

## Overview
Multi-pass multi-view PBR is a rendering approach that generates $N$ distinct perspective views of a 3D dataset by executing $N$ separate rendering passes, storing each view in a render target texture, and subsequently compositing them into a final spatial image [S1]. It is primarily used for auto-stereoscopic displays (e.g., lenticular sheets or wavelength-selective filters) requiring multiple viewpoints [S1]. It differs from single-pass multi-view techniques (such as [Half-angle slicing](1255-half-angle-slicing.md) or [Two-pass volume rendering](1292-two-pass-volume-rendering.md)) by isolating each view into its own rasterization pass, which simplifies the shader logic but incurs a significant performance penalty proportional to $N$ [S1].

## How it works
**Inputs:**
*   3D volume dataset.
*   Number of views $N$ (commonly 8).
*   Display type (lenticular sheet or wavelength-selective filter array).
*   View frustum configurations for each of the $N$ views.

**Outputs:**
*   Final spatial image $I(x, y)$ conforming to the auto-stereoscopic display device.

**Data Structures:**
*   $N$ target textures (render targets) $I_i$, where $i \in \{0, \dots, N-1\}$. Each texture holds the rendered image for one view.
*   Mask function $Mask(x, y)$, initialized once during setup based on the display type.

**Steps:**
1.  **Setup:** Initialize the mask function $Mask(x, y)$ based on the target display hardware. For lenticular sheets, $Mask(x, y) = x \mod N$. For wavelength-selective filter arrays, the mask incorporates RGB component masking and diagonal shifts [S1].
2.  **Render Views:** For each view $i$ from $0$ to $N-1$:
    a. Set the view frustum to the configuration for view $i$.
    b. Render the 3D data into target texture $I_i$ using standard rasterization or ray-splatting techniques [S1].
3.  **Composite:** Combine the $N$ target images into the final spatial image $I(x, y)$ using the mask:
    $$I(x, y) = I_{Mask(x, y)}(x, y)$$
    where $I_k(x, y)$ denotes the pixel at $(x, y)$ in texture $I_k$ [S1].

**Pitfalls:**
*   **Performance:** The method requires $N$ separate rendering passes, leading to an $N$-fold cost increase compared to single-view rendering [S1].
*   **Memory Bandwidth:** Compositing $N$ high-resolution images (e.g., 8 views of $1900 \times 1600$) requires transferring significant data (e.g., ~70MB/frame), which can bottleneck the pipeline [S1].
*   **Blurring:** If pixel-fill rate is limiting, intermediate images may be rendered at sub-resolution and up-sampled during compositing, causing blurring artifacts [S1].
*   **Geometry Overhead:** Geometry processing cost overhead remains at $N-1$ times the single-view cost, which is a major limiting factor for large datasets [S1].

## Subtopics

### Conventional Multi-View Rendering
A conventional approach that renders 3D data in $N$ passes using $N$ different perspective (off-axis asymmetric sheared) view frustum configurations, then combines the resulting $N$ images into a spatial image via masking [S1]. It is chosen when shader complexity is high or when using hardware that does not support efficient single-pass multi-view rasterization. The masking function for lenticular sheets is $Mask(x, y) = x \mod N$ [S1]. For wavelength-selective filter arrays, masking involves individual RGB component masking and diagonal shifts [S1]. It suffers from an $N$-times cost increase compared to single-view rendering [S1].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot T_{render})$, where $T_{render}$ is the time to render one view. Performance decreases by a factor of approximately 8 compared to single-view rendering for $N=8$ [S1].
*   **Memory Complexity:** Requires storage for $N$ full-resolution target images. For $N=8$ and resolution $1900 \times 1600$, this involves transferring ~70MB per frame [S1].
*   **Strengths:** Simple implementation; leverages standard render-to-texture pipelines; no complex single-pass multi-view shader logic required [S1].
*   **Limitations:** Immense performance penalty due to separate passes; geometry processing overhead scales with $N$; potential for blurring if sub-resolution rendering is used [S1].
*   **When to use:** When $N$ is small, or when the rendering pipeline cannot support efficient single-pass multi-view techniques (e.g., [Half-angle slicing](1255-half-angle-slicing.md)) [S1].
*   **When not to use:** When pixel-fill rate or memory bandwidth is a bottleneck, or when $N$ is large, as the $N$-fold cost increase becomes prohibitive [S1].

## Background
The conventional multi-view rendering approach has been used in systems such as those by SGI, Portoni et al. (2000), Schmidt and Grasnick (2002), and Miniel et al. (2004) [S1]. It was implemented and tested for 8-view auto-stereoscopic displays using wavelength-selective filter arrays with a resolution of $1900 \times 1200$ [S1]. The method was compared against single-pass vertex-based (GPU VS) and fragment-based (GPU FS) implementations, showing significant performance degradation relative to single-view rendering [S1].

## Sources
[S1] Multi-view point splatting, https://doi.org/10.1145/1174429.1174479, fetched 2026-11-03
