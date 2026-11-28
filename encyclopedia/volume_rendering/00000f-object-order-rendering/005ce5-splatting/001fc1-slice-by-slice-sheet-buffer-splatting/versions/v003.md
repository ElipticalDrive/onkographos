# Slice-by-Slice (Sheet Buffer) Splatting

Also called: Axis-Aligned Sheet-Buffer Compositing, Muller sheet-buffer, Sheet Buffer Splatting, Sheet-Buffer Method, slice-based composition

## Overview
Slice-by-Slice (Sheet Buffer) Splatting is a volume rendering optimization that improves image quality over standard "splat-every-sample" compositing by correctly handling visibility through depth-ordered accumulation [S2]. It sets itself apart from its parent method, [Splatting](23-splatting.md), by organizing splats into 2D cache-sheets aligned with the volume axis most parallel to the image plane, compositing these sheets slice-by-slice in depth order, and then blending the result into the final image [S1, S3, unofficial]. Unlike [EWA Splatting](1069-ewa-splatting.md), which handles perspective distortion per-splat, this method primarily addresses the "sparkling" artifacts of list-based accumulation by ensuring correct occlusion, though it introduces its own "popping" artifacts if the view angle crosses specific thresholds [S2][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of transformed splats (primitives) in screen space, each with position $(x, y)$, weight (opacity/density), and footprint function $F(x,y)$ [S1, S10, unofficial].
*   **Output:** A final composited image buffer.

**Data Structures:**
1.  **Sheet Buffer:** A 2D array (image-sized) aligned with the volume axis most parallel to the image plane. It stores accumulated color and opacity values for the current slice.
2.  **Cache Image:** The final output image buffer where sheet buffers are composited.
3.  **Splat List:** The collection of all splats, typically sorted or bucketed by their depth coordinate relative to the chosen compositing axis.

**Steps:**
1.  **Determine Compositing Axis:** Identify the volume axis ($x, y,$ or $z$) that is most parallel to the image plane (view direction). The sheet buffers will be aligned with this axis [S1, S3, unofficial].
2.  **Organize Splats into Sheets:** Distribute splats into cache-sheets based on their position along the compositing axis. Each sheet corresponds to a specific depth slice in the volume space [S1, S3, unofficial].
3.  **Accumulate into Sheet Buffer:** For each sheet (from back to front relative to the view):
    *   Initialize the sheet buffer with the current state of the cache image (or clear if starting a new frame) [S1, unofficial].
    *   For each splat in the current sheet:
        *   Compute the splat's contribution to the sheet buffer pixels. Since the sheet buffer is aligned with the volume axis, splat positions may need interpolation to map precisely to the slice grid if the view is not perfectly orthogonal [S1, unofficial].
        *   Rasterize the splat footprint $F(x,y)$ onto the sheet buffer.
        *   Accumulate the splat's color and opacity into the sheet buffer using alpha blending [S1, S3, unofficial].
4.  **Composite Sheet to Image:** Once all splats for a sheet are accumulated, composite the sheet buffer into the main cache image using alpha blending [S1, unofficial].
5.  **Repeat:** Process the next sheet (next depth slice) until all sheets are composited [S1, unofficial].

**Formulas:**
The accumulation in the sheet buffer follows standard front-to-back or back-to-front alpha compositing. For a pixel $p$ in the sheet buffer, the accumulated color $C_{acc}$ and opacity $\alpha_{acc}$ are updated by a splat with color $C_s$ and opacity $\alpha_s$:
$$ \alpha_{new} = \alpha_{old} + \alpha_s (1 - \alpha_{old}) $$
$$ C_{new} = \frac{C_{old} \alpha_{old} + C_s \alpha_s (1 - \alpha_{old})}{\alpha_{new}} $$
(If $\alpha_{new} = 0$, $C_{new} = C_{old}$) [S3, unofficial].

The final image pixel $C_{final}$ is obtained by compositing the sheet buffer result $C_{sheet}$ with opacity $\alpha_{sheet}$ into the existing cache image $C_{cache}$ with opacity $\alpha_{cache}$:
$$ \alpha_{final} = \alpha_{cache} + \alpha_{sheet} (1 - \alpha_{cache}) $$
$$ C_{final} = \frac{C_{cache} \alpha_{cache} + C_{sheet} \alpha_{sheet} (1 - \alpha_{cache})}{\alpha_{final}} $$

**Parameters:**
*   **Switching Threshold:** The view angle at which the compositing axis is switched to maintain alignment. Sources indicate this occurs at 45 degrees relative to the volume axes [S3, unofficial].
*   **Sheet Resolution:** Typically matches the image resolution or a lower resolution for the sheet buffer to save memory, requiring interpolation during rasterization [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Popping Artifacts:** When the view angle crosses the 45-degree threshold, the compositing axis switches abruptly (e.g., from $z$ to $x$). This causes discontinuities in the rendering order, leading to visible "popping" artifacts [S2][S3, unofficial].
*   **Interpolation Error:** Because the sheet buffer does not correspond directly to voxel positions in space (especially in perspective views), splat positions must be interpolated to lie on the slice, introducing potential precision errors [S1, unofficial].
*   **Orthographic vs. Perspective:** The method works best for orthographic views where a single footprint table and constant reconstruction kernel can be used [S1, unofficial].

## Subtopics

### [Image-Aligned Sheet-Buffer Compositing](5594-image-aligned-sheet-buffer-compositing.md)
Image-Aligned Sheet-Buffer Compositing is a variant that eliminates popping artifacts by aligning the sheet buffer compositing axis with the image plane rather than the volume axes. Unlike standard axis-aligned sheet buffers which suffer from discontinuities when the viewing angle crosses 45° relative to the data axes, this method slices interpolation kernels into sections parallel to the image plane. It sets itself apart from its parent method, [Slice-by-Slice (Sheet Buffer) Splatting](2110-slice-by-slice-sheet-buffer-splatting.md), by using image-aligned cutting planes and alpha-compositing kernel sections to ensure smooth color transitions across varying optical properties and resolutions.

### Sheet-based Splatting (Westover)
Sheet-based Splatting, proposed by Westover, is the foundational concept for sheet-buffer methods. It involves accumulating splats onto a 2D sheet aligned with the volume axis most parallel to the view plane, and then compositing these sheets in depth order into the image with a matting operation [S4, unofficial]. It sets itself apart as the original formulation of the sheet-buffer approach, distinguishing itself from pure list-based splatting by introducing depth-ordered sheet compositing to reduce sparkling artifacts [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N + S \cdot R)$, where $N$ is the number of splats, $S$ is the number of sheets, and $R$ is the resolution of the sheet buffer. It is slower than splat-every-sample composition because it avoids per-slice additions in the other method but requires explicit sheet management and compositing [S2].
*   **Memory Complexity:** $O(R^2)$ for the sheet buffer and cache image, plus $O(N)$ for the splat list.
*   **Measured Performance:** In comparisons, slice-based composition produces brighter images due to correct visibility determination but is slower than splat-every-sample composition [S2]. Specific FPS numbers vary by engine and dataset (e.g., 0.3-1.1 fps in some cited tests), but the relative performance penalty is noted [S2].
*   **Strengths:** Correct visibility determination; reduces "sparkling" artifacts associated with unsorted list-based accumulation; hardware-friendly rasterization for sheet accumulation.
*   **Limitations:** "Popping" artifacts when view angle crosses 45° threshold; high computational cost for image-aligned variants; interpolation overhead for non-orthographic views [S1, unofficial][S2][S3, unofficial].
*   **When to Use:** When image quality (correct occlusion) is prioritized over raw speed, and the view angle remains stable or within a single axis quadrant.
*   **When Not to Use:** For interactive applications with rapid camera rotation crossing 45° boundaries, or when maximum performance is required and sparkling artifacts are acceptable.

## Background
The sheet-buffer method was proposed by Müller and Yagel [Müller96] and further developed by Müller [Müller98] as an optimization for CPU-based volume ray tracing and splatting. It was introduced to address the "sparkling" artifacts of list-based splatting by organizing splats into cache-sheets aligned with the volume face most closely parallel to the image plane [S1, unofficial]. Westover also proposed similar sheet-based accumulation strategies [S4, unofficial]. The method is part of the broader family of object-order volume rendering techniques that leverage 2D rasterization hardware for compositing [S1, S3, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Hardware-Accelerated Adaptive EWA Volume Splatting, https://publications.ri.cmu.edu/storage/publications/pub_files/pub4/chen_wei_2004_1/chen_wei_2004_1.pdf, fetched 2026-10-27
[S3] Volumetric Points - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/teaching/cse616/volPts.pdf, fetched 2026-10-29 (unofficial)
[S4] DTIC ADA606342: An Anti-Aliasing Technique for Splatting, https://archive.org/details/DTIC_ADA606342, fetched 2026-10-26 (unofficial)
