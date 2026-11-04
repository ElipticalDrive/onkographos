# Slice-by-Slice (Sheet Buffer) Splatting

Also called: Axis-Aligned Sheet-Buffer Compositing, Muller sheet-buffer, Sheet Buffer Splatting, Sheet-Buffer Method, slice-based composition

## Overview
Slice-by-Slice (Sheet Buffer) Splatting is a compositing strategy that organizes splats into cache-sheets aligned parallel to the volume face most closely parallel to the image plane, then composites these sheets slice-by-slice [S1, S3, unofficial]. It solves the visibility determination errors inherent in "splat-every-sample" composition by ensuring correct depth ordering within each sheet, producing brighter and more accurate images [S2]. It is distinguished from its siblings by its use of axis-aligned accumulation buffers that are composited in depth order, whereas [EWA Splatting](1069-ewa-splatting.md) typically composites splats directly into the image or uses different footprint approximations, and [Ray-Driven Splatting](1156-ray-driven-splatting.md) composites along rays [S3, unofficial].

## How it works
**Inputs:** Volume data, viewing parameters (camera position, orientation, projection matrix), transfer function.
**Outputs:** Final 2D image.

**Data Structures:**
1.  **Sheet Buffer:** A 2D image buffer (accumulation buffer) aligned with the current compositing axis (e.g., XY, XZ, or YZ). It stores intermediate color and opacity values.
2.  **Cache Image:** The final output image buffer.
3.  **Voxel List:** Sorted by depth relative to the current sheet orientation.

**Steps:**
1.  **Sheet Selection:** Determine the volume axis (X, Y, or Z) whose slices are most parallel to the image plane. Switch the compositing axis when the view angle crosses a 45-degree threshold relative to the volume axes to maintain alignment [S3, unofficial].
2.  **Sorting:** Sort the volume voxels (or splats) in back-to-front order relative to the current view direction [S1, unofficial].
3.  **Sheet Processing Loop:**
    a.  Initialize the current sheet buffer to transparent.
    b.  Iterate through voxels belonging to the current slice (sheet) of the selected axis.
    c.  **Projection:** For each voxel, compute its 2D screen position. Since the sheet buffer is aligned with the volume axis and not necessarily the image plane, interpolation is required to map voxel contributions to the sheet buffer coordinates [S1, unofficial].
    d.  **Splatting:** Project the 3D reconstruction kernel onto the sheet buffer. For regular grids, the footprint can be approximated using look-up tables if the view is orthographic [S1, unofficial].
    e.  **Accumulation:** Add the voxel's contribution (color and opacity) to the sheet buffer at the projected location.
4.  **Sheet Compositing:** Once all voxels in the current slice have been processed into the sheet buffer, composite the sheet buffer into the cache image (final image) using alpha blending [S1, unofficial].
5.  **Advance:** Move to the next slice in the back-to-front order and repeat steps 3-4 until all slices are processed [S1, unofficial].

**Formulas:**
The contribution of a voxel $V$ to the sheet buffer at position $(u,v)$ is:
$$ C_{sheet}(u,v) = C_{sheet}(u,v) \cdot (1 - \alpha_V) + C_V \cdot \alpha_V $$
$$ \alpha_{sheet}(u,v) = \alpha_{sheet}(u,v) + \alpha_V \cdot (1 - \alpha_{sheet}(u,v)) $$
where $C_V$ and $\alpha_V$ are the color and opacity of the voxel's splat at $(u,v)$ [S3, unofficial].

**Parameters:**
*   **Switching Threshold:** 45 degrees between volume axes [S3, unofficial].
*   **Kernel Radius:** Typically 2.0 standard deviations for Gaussian kernels [S10, unofficial].

**Edge Cases and Pitfalls:**
*   **Popping Artifacts:** Sudden changes in the compositing axis (e.g., crossing 45 degrees) cause visible "popping" artifacts because the accumulation buffer alignment shifts abruptly [S1, unofficial][S2][S3, unofficial].
*   **Interpolation Error:** Since the sheet buffer does not correspond directly to voxel positions in space (especially in perspective views), interpolation is needed to place splats correctly, which can introduce blurring or misalignment if not handled precisely [S1, unofficial].
*   **Performance:** This method is slower than splat-every-sample composition because it requires per-slice additions and compositing steps [S2].

## Subtopics

### Image-Aligned Sheet-Buffer Compositing
A variant proposed by Mueller [S3, unofficial] that eliminates popping artifacts by aligning the sheet buffer with the image plane rather than the volume axes. It involves slicing the kernel into sections that are image-aligned, requiring multiple footprint rasterizations per point [S3, unofficial]. It is chosen when artifact-free animation is critical, despite its higher computational cost [S2][S3, unofficial].

## Operating characteristics
**Time Complexity:** Higher than splat-every-sample due to the overhead of managing sheet buffers and compositing them slice-by-slice [S2].
**Memory Complexity:** Requires additional memory for the sheet buffer(s) and cache image [S1, unofficial].
**Performance:** In benchmarks, slice-based composition produces brighter images due to correct visibility but is slower. For example, on the "Bonsai" dataset, traditional pre-classification achieved 7.23 fps while a list-based approach achieved 0.8 fps, though direct comparison to slice-based is complex due to implementation differences [S2].
**Strengths:** Correct visibility determination; brighter, more accurate images than splat-every-sample [S2].
**Limitations:** Popping artifacts at axis-switching thresholds; higher computational cost; interpolation requirements for non-orthographic views [S1, unofficial][S2][S3, unofficial].
**When to use:** When image accuracy and correct visibility are prioritized over maximum frame rate, and popping artifacts can be mitigated by smooth camera motion or image-aligned variants [S2][S3, unofficial].

## Background
The method was proposed by Mueller and Crawfis [S2][S3, unofficial] as an optimization to mitigate popping artifacts associated with axis-aligned sheet buffers, though their initial image-aligned kernel-slicing approach had high computational cost [S2]. It builds on the shared principles of splatting [S10, unofficial], specifically the sheet-based traversal and compositing steps [S10, unofficial]. It is primarily used in CPU-based volume rendering and early GPU implementations where cache coherence and memory bandwidth were significant constraints [S1, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Hardware-Accelerated Adaptive EWA Volume Splatting, https://publications.ri.cmu.edu/storage/publications/pub_files/pub4/chen_wei_2004_1/chen_wei_2004_1.pdf, fetched 2026-10-27
[S3] Volumetric Points - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/teaching/cse616/volPts.pdf, fetched 2026-10-29 (unofficial)
