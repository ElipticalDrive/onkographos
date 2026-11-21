# Image-Aligned Sheet-Buffer Compositing


## Overview
Image-aligned sheet-buffer compositing is a volume rendering technique that eliminates popping artifacts by aligning compositing sheets with the image plane regardless of viewing direction [S1, unofficial]. Unlike [Axis-Aligned Sheet-Buffered Splatting](5593-axis-aligned-sheet-buffered-splatting.md), which organizes voxels into fixed volume slices, this method advances a slab across the volume, slicing kernels into sections and projecting them into image-aligned sheet buffers [S1, unofficial].

## How it works
**Inputs:** Volume data, viewing parameters, sheet buffer resolution [G].
**Outputs:** Final composited image [G].

**Data Structures:**
*   **Sheet Buffer:** A 2D buffer aligned with the image plane, storing accumulated color and opacity for each pixel [G]. Exact layout, fields, sizes, and bit packing: Not given in the sources [G].
*   **Compositing Buffer:** The final image buffer where sheet buffers are composited [S1, unofficial].

**Steps:**
1.  **Slab Advancement:** Advance a slab across the volume so that the slab cuts kernels into sections [S1, unofficial].
2.  **Footprint Mapping:** For each voxel or point within the current slab, perform footprint mapping as usual [S1, unofficial]. This requires multiple footprint rasterizations per point to account for the intersection of the kernel with the image-aligned sheet [S1, unofficial].
3.  **Kernel Section Addition:** Add the resulting kernel sections into the sheet buffer [S1, unofficial].
4.  **Compositing:** Composite the sheet buffers into the final image [S1, unofficial].

**Formulas:** Not given in the sources [G].
**Parameters:** Not given in the sources [G].
**Edge Cases and Pitfalls:**
*   **Popping Artifacts:** Axis-aligned methods suffer from popping when the viewing angle crosses thresholds near 45° (e.g., 44.7° vs 45.2°) [S1, unofficial]. Image-aligned methods eliminate this by aligning with the image plane [S1, unofficial].
*   **Computational Cost:** Requires multiple footprint rasterizations per point, increasing computational expense [S1, unofficial].

## Subtopics

### [Image-aligned Splatting](13640-image-aligned-splatting.md)
A variant that aligns compositing sheets with the image plane regardless of viewing direction, eliminating popping artifacts. It sets itself apart from axis-aligned sheet-buffered splatting by advancing a slab across the volume rather than using fixed volume slices. Choose it when accurate voxel field reconstruction and smooth anti-aliasing are required, accepting higher computational cost due to multiple splatting of voxels.

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources [G].
*   **Measured Performance:** Not given in the sources [G].
*   **Scaling:** Not given in the sources [G].
*   **Strengths:** Eliminates popping artifacts [S1, unofficial].
*   **Limitations:** Higher computational expense due to multiple splatting of individual voxels [S1, unofficial]. Requires multiple footprint rasterizations per point [S1, unofficial].
*   **When to use:** Ideal for rendering where popping artifacts must be avoided [S1, unofficial].
*   **When not to use:** When computational resources are limited [G].

## Background
Originated from work by Mueller ('98) on volumetric points and sheet buffers [S1, unofficial]. The method addresses limitations of axis-aligned splatting, such as popping artifacts observed at specific viewing angles (e.g., 44.7° vs 45.2°) [S1, unofficial].

## Sources
[S1] Volumetric Points - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/teaching/cse616/volPts.pdf, fetched 2026-10-29 (unofficial)
