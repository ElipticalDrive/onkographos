# Slice-by-Slice (Sheet Buffer) Splatting

Also called: Axis-Aligned Sheet-Buffer Compositing, Muller sheet-buffer, Sheet Buffer Splatting, Sheet-Buffer Method, slice-based composition

## Overview
Slice-by-Slice (Sheet Buffer) Splatting is a compositing optimization that organizes splats into cache-sheets aligned parallel to the volume face most closely parallel to the image plane, then composites them slice-by-slice [S1, S3, unofficial]. It solves the visibility determination errors inherent in "splat-every-sample" composition by ensuring correct front-to-back accumulation within each sheet, producing brighter and more accurate images [S2]. It differs from [EWA Splatting](1069-ewa-splatting.md) by prioritizing cache-coherent sheet traversal over strict anisotropic footprint mapping, and from [Ray-Driven Splatting](1156-ray-driven-splatting.md) by leveraging hardware rasterization for bulk sheet compositing rather than per-ray sampling [S1, S3, unofficial].

## How it works
**Inputs:** Volume data, transfer function, camera parameters (view and projection matrices), and a reconstruction kernel $h$.
**Outputs:** Final 2D image.

**Data Structures:**
1.  **Sheet Buffer:** A 2D buffer aligned with the image plane (or the volume axis most parallel to it). It stores accumulated color and opacity for the current slice.
2.  **Cache Image:** The accumulating result buffer.
3.  **Splat List:** Primitives sorted by depth relative to the current sheet orientation.

**Steps:**
1.  **Sheet Orientation Selection:** Determine the volume axis (X, Y, or Z) most parallel to the image plane. The sheet buffers are aligned parallel to this axis to minimize popping artifacts [S1, S3, unofficial].
2.  **Splat Organization:** Organize splats into groups (sheets) corresponding to the slices along the chosen axis. For each splat $k$, compute its projected position on the sheet buffer. Since the sheet buffer does not correspond directly to voxel positions in space, new positions must be computed using interpolation so they lie correctly on the slice [S1, unofficial].
3.  **Rasterization (Footprint Mapping):** Project the 3D kernel footprint onto the 2D sheet buffer. For axis-aligned sheets, this typically requires a single footprint table if the view is orthographic [S1, unofficial]. For perspective views or image-aligned sheets, multiple footprint rasterizations per point may be required [S3, unofficial].
4.  **Sheet Compositing:** Traverse the volume back-to-front (or front-to-back, depending on the blending equation) slice-by-slice. For each slice:
    *   Add voxel contributions to the current sheet buffer.
    *   Composite the sheet buffer into the cache image.
    *   Process the next slice [S1, unofficial].
5.  **Switching Threshold:** To maintain alignment with the view and avoid popping, switch the compositing axis when the view angle crosses 45 degrees relative to the volume axes [S3, unofficial].

**Formulas:**
The contribution of a voxel $D$ to a screen point $(x,y)$ is determined by the footprint function $F(x,y)$, which is the integral of the 3D kernel along the view direction:
$$ F(x,y) = \int h(x-x_D, y-y_D, z) \, dz $$
The intensity written to the pixel is modulated by the voxel density:
$$ P(x,y) = \text{weight}(x,y) \cdot \rho(D) $$
where $\text{weight}(x,y) = F(x-x_D, y-y_D)$ [S10, unofficial].

**Parameters:**
*   **Switching Angle:** 45 degrees. Switching occurs when the view direction crosses this threshold relative to the volume axes [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Popping Artifacts:** Sudden changes in sheet orientation cause visible popping. This occurs when the view angle crosses the 45-degree switching threshold (e.g., from 44.7° to 45.2°) [S2][S3, unofficial].
*   **Interpolation Errors:** Because the sheet buffer does not correspond to voxel positions in space, interpolation is required for position mapping. Inaccurate interpolation leads to artifacts [S1, unofficial].
*   **Orthographic vs. Perspective:** The method works best for orthographic views as it requires only a single footprint table and a constant reconstruction kernel [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def slice_by_slice_splatting(volume, camera, kernel):
    # Determine the axis most parallel to the image plane
    axis = get_most_parallel_axis(volume, camera.view_direction)
    
    # Initialize cache image
    cache_image = zeros(image_width, image_height, 4)
    
    # Sort splats by depth along the chosen axis
    splats = sort_splats_by_axis(volume.splats, axis)
    
    # Group splats into sheets (slices)
    sheets = group_splats_into_sheets(splats, axis)
    
    for sheet in sheets:
        # Create a temporary sheet buffer aligned with the image plane
        sheet_buffer = zeros(image_width, image_height, 4)
        
        for splat in sheet:
            # Compute interpolated position on the sheet buffer
            # since the buffer does not correspond to voxel positions in space
            screen_pos = interpolate_splat_position(splat, axis, camera)
            
            # Rasterize the footprint onto the sheet buffer
            rasterize_footprint(splat, kernel, screen_pos, sheet_buffer)
        
        # Composite the sheet buffer into the cache image
        # Traverse back-to-front for correct transparency
        composite_sheet_to_cache(sheet_buffer, cache_image)
        
    return cache_image
```

## Subtopics

### Image-Aligned Sheet-Buffer Compositing
A variant proposed by Müller [S3, unofficial] that eliminates popping artifacts by aligning the sheet buffer strictly with the image plane, regardless of the volume axes. It involves slicing the kernel slab into sections, performing footprint mapping (which may require multiple footprint rasterizations per point), adding kernel sections into the sheet buffer, and then compositing the sheet buffers [S3, unofficial]. This approach has a high computational cost compared to axis-aligned methods [S2].

## Operating characteristics
*   **Time Complexity:** Slower than "splat-every-sample" composition because it avoids per-slice additions in the other method but requires additional sheet management and compositing steps [S2].
*   **Memory Complexity:** Requires additional memory for sheet buffers and cache images.
*   **Performance:** Produces brighter images due to correct visibility determination, but is slower [S2].
*   **Strengths:** Correct visibility determination; reduced sparkling compared to splat-every-sample [S2].
*   **Limitations:** Popping artifacts when view angle crosses 45-degree threshold; high computational cost for image-aligned variants [S2][S3, unofficial].
*   **When to use:** When correct visibility is critical and popping artifacts are acceptable or mitigated by smooth camera motion.
*   **When not to use:** When real-time performance is paramount and slight visibility errors are acceptable (use splat-every-sample instead) [S2].

## Background
The method was proposed by Müller and Crawfis [S2][S3, unofficial] as an optimization to mitigate popping artifacts associated with axis-aligned sheet buffers. Müller and Yagel [S1, unofficial] suggested a hybrid method where voxel contributions are partly precomputed. The approach is particularly effective for orthographic views [S1, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Hardware-Accelerated Adaptive EWA Volume Splatting, https://publications.ri.cmu.edu/storage/publications/pub_files/pub4/chen_wei_2004_1/chen_wei_2004_1.pdf, fetched 2026-10-27
[S3] Volumetric Points - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/teaching/cse616/volPts.pdf, fetched 2026-10-29 (unofficial)
