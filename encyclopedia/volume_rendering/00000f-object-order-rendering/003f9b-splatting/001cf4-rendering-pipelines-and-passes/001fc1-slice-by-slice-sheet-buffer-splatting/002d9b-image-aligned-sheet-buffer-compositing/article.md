# Image-Aligned Sheet-Buffer Compositing

Also called: Image-aligned splatting, Image-aligned sheet-buffers

## Overview
Image-aligned sheet-buffer compositing is a volume rendering technique that eliminates popping artifacts and color bleeding by maintaining compositing sheets aligned with the image plane regardless of the viewing direction [S1, unofficial]. Unlike axis-aligned sheet-buffered splatting, which organizes voxels into fixed volume slices, this method advances a slab across the volume, slicing and projecting all intersecting kernels into the sheet buffers [S1, unofficial]. It provides the most accurate reconstruction of the voxel field prior to compositing, making it ideal for post-classified rendering [S1, unofficial].

## How it works
**Inputs:** Volume data (e.g., binary cube or density field), viewing parameters (camera position, orientation), sheet buffer resolution.
**Outputs:** Final composited image.

**Data Structures:**
*   **Sheet Buffer:** A 2D buffer aligned with the image plane, storing accumulated color and opacity for each pixel.
*   **Compositing Buffer:** The final image buffer where sheet buffers are composited [S1, unofficial].

**Steps:**
1.  **Slab Advancement:** Advance a slab across the volume. The slab is defined to cut kernels into sections [S1, unofficial].
2.  **Footprint Mapping:** For each voxel or point within the current slab, perform footprint mapping as usual [S1, unofficial]. This requires multiple footprint rasterizations per point to account for the intersection of the kernel with the image-aligned sheet [S1, unofficial].
3.  **Kernel Section Addition:** Add the resulting kernel sections into the sheet buffer [S1, unofficial]. This step accumulates the contribution of the sliced kernels into the image-aligned buffers.
4.  **Compositing:** Composite the sheet buffers into the final image [S1, unofficial]. This involves blending the accumulated values in the sheet buffers to produce the final pixel colors.

**Edge Cases and Pitfalls:**
*   **Popping Artifacts:** Axis-aligned methods suffer from popping when the viewing angle crosses certain thresholds (e.g., near 45°) [S1, unofficial]. Image-aligned methods eliminate this by aligning with the image plane [S1, unofficial].
*   **Computational Cost:** Requires multiple footprint rasterizations per point, increasing computational expense [S1, unofficial].

## Subtopics

### [Image-aligned Splatting](13640-image-aligned-splatting.md)
Eliminates popping artifacts and color bleeding by maintaining compositing sheets aligned with the image plane regardless of the viewing direction. Unlike axis-aligned sheet-buffered splatting, which organizes voxels into fixed volume slices, this method advances a slab across the volume, slicing and projecting all intersecting kernels into the sheet buffers. It provides the most accurate reconstruction of the voxel field prior to compositing, making it ideal for post-classified rendering, at the cost of higher computational expense due to multiple splatting of individual voxels.

## Operating characteristics
*   **Strengths:** Eliminates popping artifacts and color bleeding [S1, unofficial]. Provides accurate reconstruction of the voxel field prior to compositing [S1, unofficial].
*   **Limitations:** Higher computational expense due to multiple splatting of individual voxels [S1, unofficial]. Requires multiple footprint rasterizations per point [S1, unofficial].
*   **When to use:** Ideal for post-classified rendering where accuracy is paramount [S1, unofficial].
*   **When not to use:** When computational resources are limited and popping artifacts are acceptable [S1, unofficial].

## Background
Originated from work by Mueller ('98) on volumetric points and sheet buffers [S1, unofficial]. The method addresses limitations of axis-aligned splatting, such as popping artifacts observed at specific viewing angles (e.g., 44.7° vs 45.2°) [S1, unofficial]. It is part of the broader family of pre-classified and post-classified rendering techniques [S1, unofficial].

## Sources
[S1] Volumetric Points - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/teaching/cse616/volPts.pdf, fetched 2026-10-29 (unofficial)
