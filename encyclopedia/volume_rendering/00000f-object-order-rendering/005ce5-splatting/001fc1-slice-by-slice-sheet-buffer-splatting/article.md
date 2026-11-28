# Slice-by-Slice (Sheet Buffer) Splatting

Also called: Axis-Aligned Sheet-Buffer Compositing, Muller sheet-buffer, Sheet Buffer Splatting, Sheet-Buffer Method, slice-based composition

## Overview
Slice-by-Slice (Sheet Buffer) Splatting is a volume rendering optimization that organizes splats into cache-sheets aligned parallel to the volume face most closely parallel to the image plane, then composites these sheets slice-by-slice [S1, unofficial][S3, unofficial]. It sets itself apart from ray-casting and screen-quad methods by compositing splats into aligned sheet buffers rather than sampling along rays or rendering full-screen quads, trading the "sparkling" of per-splat compositing for "popping" artifacts when the view crosses 45° relative to the data axes [S2][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume dataset (regular grid), camera parameters (view matrix, projection matrix), transfer function [G].
*   **Output:** 2D image (color and alpha buffer) [G].

**Data Structures:**
1.  **Sheet Buffer:** A 2D array (texture or render target) matching the screen resolution that accumulates the weighted contributions of splats belonging to a specific depth slice [S1, unofficial].
2.  **Cache Image:** The current accumulated result of composited slices [S1, unofficial].
3.  **Splat List:** A list of transformed splats, typically sorted or bucketed by their depth coordinate along the alignment axis [G].

**Method Steps:**
1.  **Sheet Alignment:** Determine the volume axis ($x, y,$ or $z$) most parallel to the image plane. Switch the compositing axis at 45° to maintain alignment with the view [S3, unofficial].
2.  **Splat Transformation:** Transform splat centers from volume space to screen space. Compute new positions using interpolation so that they lie on the slice, since the buffer does not correspond to voxel positions in space [S1, unofficial].
3.  **Bucketing:** Assign each splat to a specific sheet buffer corresponding to its depth slice index $k$ along the alignment axis [S1, unofficial].
4.  **Rasterization:** Rasterize all splats assigned to sheet $k$ onto the sheet buffer. The contribution of a voxel $D$ to a screen point $(x,y)$ is determined by the footprint function $F(x,y)$, which is the integral of the 3D kernel along the view direction [G]:
    $$ F(x,y) = \int h(x-x_D, y-y_D, z) \, dz $$
    The intensity written to the pixel is modulated by the voxel density:
    $$ P(x,y) = \text{weight}(x,y) \cdot \rho(D) $$
    where $\text{weight}(x,y) = F(x-x_D, y-y_D)$ [G].
5.  **Compositing:** Traverse the sheet buffers in depth order (back-to-front for opacity) [S3, unofficial]. For each sheet buffer:
    *   Composite the sheet buffer into the cache image using alpha blending [S1, unofficial][S2][S3, unofficial].
    *   This step effectively adds voxel contributions slice-by-slice [S1, unofficial].
    *   Whenever a sheet buffer has received all contributions, it is composited with the current image and the next slice is processed [S1, unofficial].
6.  **Output:** The cache image is the final rendered image [G].

**Parameters:**
*   **Switching Threshold:** The view angle at which the alignment axis switches (typically 45°) [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Popping Artifacts:** Occur when the orientation of the sheets suddenly changes, specifically when the view angle crosses the 45-degree switching threshold (e.g., from 44.7° to 45.2°) [S1, unofficial][S2][S3, unofficial].
*   **Interpolation:** Since the sheet buffer does not correspond directly to voxel positions in space, new positions for splats must be computed using interpolation to lie on the slice [S1, unofficial].
*   **Orthographic vs. Perspective:** The method works best for orthographic views because it requires only a single footprint table and reconstruction kernel, which is constant except for screen space offset [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def slice_by_slice_splatting(volume, camera, image_width, image_height):
    # 1. Determine alignment axis based on camera view direction
    # Align with volume axis most parallel to image plane [G]
    view_dir = normalize(camera.view_matrix[2][:3])
    axes = [vec(1,0,0), vec(0,1,0), vec(0,0,1)]
    alignment_axis = max(axes, key=lambda a: abs(dot(view_dir, a)))
    
    # 2. Initialize sheet buffers and cache image
    # Sheet buffers are 2D arrays matching screen resolution [G]
    num_slices = volume.shape[alignment_axis]
    sheet_buffers = [create_zero_2d_array(image_width, image_height) for _ in range(num_slices)]
    cache_image = create_zero_2d_array(image_width, image_height)
    
    # 3. Transform and bucket splats
    splats = transform_splats(volume, camera)
    bucket_splats_into_sheets(splats, sheet_buffers, alignment_axis)
    
    # 4. Rasterize splats into sheet buffers
    for k, sheet in enumerate(sheet_buffers):
        clear_sheet(sheet)
        for splat in sheet.splats:
            # Compute footprint F(x,y) and weight [G]
            footprint = compute_footprint(splat.kernel, splat.center, camera)
            weight = footprint.evaluate(splat.center_screen)
            color = splat.color * weight
            
            # Rasterize onto sheet buffer
            rasterize_splat(sheet, splat, color)
            
    # 5. Composite sheets into cache image
    # Traverse in depth order (back-to-front for standard alpha blending) [G]
    sorted_indices = sort_sheets_by_depth(sheet_buffers, alignment_axis, camera)
    for k in sorted_indices:
        composite_sheet_into_image(sheet_buffers[k], cache_image)
        
    return cache_image

def determine_alignment_axis(view_matrix):
    # Returns 'x', 'y', or 'z' based on which volume axis is most parallel to the image plane [G]
    view_dir = normalize(view_matrix[2][:3])
    axes = [vec(1,0,0), vec(0,1,0), vec(0,0,1)]
    return ['x', 'y', 'z'][max(range(3), key=lambda i: abs(dot(view_dir, axes[i])))]

def transform_splats(volume, camera):
    # Applies MVP matrix to volume centers, computes screen positions and projected kernel parameters [G]
    splats = []
    for voxel in volume:
        center_3d = voxel.position
        center_4d = vec4(center_3d, 1.0)
        projected = camera.mvp_matrix @ center_4d
        center_screen = projected.xy / projected.w
        kernel = project_kernel(voxel.kernel, camera)
        splats.append({'center': center_3d, 'center_screen': center_screen, 'kernel': kernel, 'color': voxel.color})
    return splats

def bucket_splats_into_sheets(splats, sheet_buffers, axis):
    # Assigns each splat to sheet_buffers[depth_index] along the chosen axis [G]
    for splat in splats:
        depth_idx = int(round(splat['center'][axis] / volume.voxel_size[axis]))
        if 0 <= depth_idx < len(sheet_buffers):
            sheet_buffers[depth_idx].splats.append(splat)

def compute_footprint(kernel, center, camera):
    # Computes 2D footprint function F(x,y) by integrating 3D kernel along view direction [G]
    # For orthographic views, footprint is constant per kernel type
    return kernel.integrate_along_view(camera.projection_type)

def rasterize_splat(sheet, splat, color):
    # Rasterizes the footprint of the splat onto the 2D sheet buffer
    bounds = splat.kernel.bounding_box_2d(splat['center_screen'])
    for y in range(bounds.min_y, bounds.max_y):
        for x in range(bounds.min_x, bounds.max_x):
            if 0 <= x < sheet.width and 0 <= y < sheet.height:
                alpha = splat.kernel.evaluate(vec2(x, y) - splat['center_screen'])
                sheet[y, x] = blend(sheet[y, x], color, alpha)

def sort_sheets_by_depth(sheets, axis, camera):
    # Returns indices of sheets sorted back-to-front relative to view [G]
    sheet_depths = [(i, sheet.splats[0]['center'][axis] if sheet.splats else 0) for i, sheet in enumerate(sheets)]
    return [idx for idx, _ in sorted(sheet_depths, key=lambda item: item[1], reverse=True)]

def composite_sheet_into_image(sheet, cache_image):
    # Blends the accumulated sheet buffer into the main cache image
    for y in range(sheet.height):
        for x in range(sheet.width):
            pixel_color, pixel_alpha = sheet[y, x]
            cache_image[y, x] = blend(cache_image[y, x], (pixel_color, pixel_alpha), 1.0)

def blend(src, dst, alpha):
    # Standard alpha compositing: out = src * (1 - alpha) + dst * alpha [G]
    return src * (1.0 - alpha) + dst * alpha
```

## Subtopics

### [Image-Aligned Sheet-Buffer Compositing](5594-image-aligned-sheet-buffer-compositing.md)
Image-Aligned Sheet-Buffer Compositing is a variant that eliminates "popping" artifacts by aligning the sheet buffer compositing axis with the image plane, rather than the volume axes. Unlike standard axis-aligned sheet buffers, which suffer from discontinuities when the viewing angle crosses 45° relative to the data axes, this method slices interpolation kernels into sections parallel to the image plane. It sets itself apart from its parent method, [Slice-by-Slice (Sheet Buffer) Splatting](2110-slice-by-slice-sheet-buffer-splatting.md), by using image-aligned cutting planes and alpha-compositing kernel sections to ensure smooth color transitions across varying optical properties and resolutions. However, it has a high computational cost and requires multiple footprint rasterizations per point.

### Sheet-based Splatting (Westover)
Sheet-based Splatting, proposed by Westover, accumulates splats onto a 2D sheet aligned with the volume axis most parallel to the view plane, then composites the sheets in depth order into the image with a matting operation [S4, unofficial]. It is a foundational concept for the sheet-buffer approach, improving image quality over simple splat-every-sample composition by handling depth ordering more effectively [S4, unofficial].

## Operating characteristics
*   **Performance:** Slice-based composition produces brighter images due to correct visibility determination but is slower than splat-every-sample composition because it avoids per-slice additions in the other method [S2].
*   **Strengths:** Hardware-accelerated, good image quality compared to simple splatting, avoids sparkling artifacts [S3, unofficial].
*   **Limitations:** Popping artifacts at 45° view angles, high computational cost for image-aligned variants, best suited for orthographic views [S1, unofficial][S2][S3, unofficial].
*   **When to use:** When real-time performance is needed and the view angle is stable or orthographic [S1, unofficial].
*   **When not to use:** When the view angle frequently crosses 45° thresholds and popping artifacts are unacceptable [S2][S3, unofficial].

## Background
The sheet-buffer method was proposed by Müller and Yagel [Müller96] and Müller [Müller98] as an optimization for CPU-based volume rendering [S1, unofficial]. It builds on Westover's earlier work on sheet-based accumulation [S4, unofficial]. Müller and Crawfis later proposed image-aligned kernel-slicing to mitigate popping artifacts, though this increased computational cost [S2].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Hardware-Accelerated Adaptive EWA Volume Splatting, https://publications.ri.cmu.edu/storage/publications/pub_files/pub4/chen_wei_2004_1/chen_wei_2004_1.pdf, fetched 2026-10-27
[S3] Volumetric Points - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/teaching/cse616/volPts.pdf, fetched 2026-10-29 (unofficial)
[S4] DTIC ADA606342: An Anti-Aliasing Technique for Splatting, https://archive.org/details/DTIC_ADA606342, fetched 2026-10-26 (unofficial)
