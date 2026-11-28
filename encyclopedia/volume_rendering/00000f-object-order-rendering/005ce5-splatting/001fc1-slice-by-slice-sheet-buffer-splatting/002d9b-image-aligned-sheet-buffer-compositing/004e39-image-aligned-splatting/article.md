# Image-aligned Splatting

Also called: Axis-Aligned Sheet-Buffered Splatting, Image-Aligned Sheet-Buffered Splatting, Image-aligned Splatting with Alpha Composition, Kernel section composition, Mueller 76, Sheet-bucket splatting, slab-based splatting

## Overview
Image-aligned splatting is a volume rendering technique that eliminates color bleeding and popping artifacts by aligning the compositing sheet with the image plane at all times, regardless of the viewing direction [S2, unofficial][S3, unofficial]. Unlike its sibling [Axis-Aligned Sheet-Buffered Splatting](5593-axis-aligned-sheet-buffered-splatting.md), which organizes voxels into fixed volume slices parallel to the viewing axis, image-aligned splatting advances a slab across the volume, slicing and projecting all intersecting kernels into sheet buffers [S2, unofficial][S4, book]. This approach provides the most accurate reconstruction of the voxel field prior to compositing and produces smoother images with better anti-aliasing characteristics than raycasting, though at a higher computational cost due to multiple splatting of voxels [S2, unofficial][S4, book].

## How it works
**Inputs:** Volume dataset containing density, color, and opacity fields [S2, unofficial][S4, book]. Viewing parameters including viewpoint, direction, and projection matrix [S3, unofficial]. Slab width $\Delta s$ [S3, unofficial]. Kernel base distribution and scaling factor [S1, unofficial].
**Outputs:** Final composited image buffer.

**Data Structures:**
1. **Sheet Buffer:** A 2D buffer aligned with the image plane, storing accumulated color $C$ and opacity $\alpha$ for the current slab [S2, unofficial][S4, book].
2. **Occlusion Map:** A dynamically computed screen-space map used for early splat elimination, updated via convolution after each sheet composite [S3, unofficial].
3. **Kernel Sections:** Pre-integrated footprints of voxel kernels, stacked to form a single kernel representation [S1, unofficial][S3, unofficial].

**Steps:**
1. **Kernel Preparation:** Compute kernel sections using a kernel base distribution and a scaling factor [S1, unofficial]. Stack sections to form a single kernel representation [S1, unofficial].
2. **Slab Advancement:** Define a slab of width $\Delta s$ aligned parallel to the image plane [S3, unofficial].
3. **Voxel Selection and Transformation:** Identify all voxel kernels that overlap the current slab [S3, unofficial]. Transform the center of each relevant voxel into screen space [S3, unofficial].
4. **Early Splat Elimination:** Index the occlusion map using the transformed screen coordinates to test for visibility [S3, unofficial]. If the voxel is occluded, skip it, noting that transformation is still required even if the voxel is culled [S3, unofficial].
5. **Clipping and Projection:** For visible voxels, clip the voxel kernels to the slab boundaries [S3, unofficial]. Slice and project the intersecting kernels [S2, unofficial].
6. **Sheet Buffer Accumulation:** Sum the clipped kernels into the sheet buffer [S3, unofficial]. All splats within a single sheet are added together; they are not composited within the sheet [S2, unofficial].
7. **Sheet Composition:** Composite the current sheet buffer with the previous accumulated result [S3, unofficial].
8. **Occlusion Map Update:** Perform a convolution operation on the occlusion map to update it based on the current sheet's contributions [S3, unofficial]. This operation can be limited to buffer tiles that received splat contributions in the current slab to save time [S3, unofficial].
9. **Iteration:** Advance the slab across the volume and repeat steps 2–8 until the volume is fully processed [S2, unofficial].

**Formulas:**
The color, opacity, and feature terms are calculated as averages over the slab interval $\Delta s$ [S3, unofficial]:

$$
C_\lambda(s_i) = \frac{\int_{i\Delta s}^{(i+1)\Delta s} C_\lambda(s) ds}{\Delta s}
$$

$$
\alpha(s_i) = \frac{\int_{i\Delta s}^{(i+1)\Delta s} \alpha(s) ds}{\Delta s}
$$

$$
f(s_i) = \frac{\int_{i\Delta s}^{(i+1)\Delta s} f(s) ds}{\Delta s}
$$

Where $s_i$ represents the slab index, and $C_\lambda$, $\alpha$, and $f$ are the color, opacity, and feature values respectively [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def image_aligned_splatting(volume, view_params, slab_width_delta_s, kernel_base, scaling_factor):
    # Initialize final composited image and opacity buffers [G]
    final_image = [[0.0, 0.0, 0.0] for _ in range(view_params.image_height)]
    final_opacity = [[0.0] for _ in range(view_params.image_height)]
    
    # Initialize occlusion map for early splat elimination [G]
    occlusion_map = [[0.0] for _ in range(view_params.image_height)]
    
    # Pre-compute kernel sections and stack them [S1, unofficial]
    kernel_sections = compute_kernel_sections(kernel_base, scaling_factor)
    
    # Determine slab range along the view direction [G]
    slab_start = volume.bounds.min_z
    slab_end = volume.bounds.max_z
    current_slab_z = slab_start
    
    while current_slab_z < slab_end:
        slab = Slab(z_min=current_slab_z, z_max=current_slab_z + slab_width_delta_s)
        current_sheet = [[0.0, 0.0, 0.0] for _ in range(view_params.image_height)]
        current_sheet_opacity = [[0.0] for _ in range(view_params.image_height)]
        
        # Identify voxels intersecting the current slab [S3, unofficial]
        for voxel in volume.voxels:
            if not voxel.intersects_slab(slab):
                continue
            
            # Transform voxel center to screen space [S3, unofficial]
            screen_pos = transform_to_screen(voxel.center, view_params)
            
            # Early splat elimination: test visibility against occlusion map [S3, unofficial]
            if not is_visible(screen_pos, occlusion_map):
                continue
            
            # Clip kernel to slab boundaries and project [S3, unofficial]
            clipped_kernel = clip_kernel_to_slab(voxel.kernel, slab)
            rasterize_footprint(current_sheet, current_sheet_opacity, clipped_kernel, screen_pos)
        
        # Composite current sheet with previous accumulated result [S3, unofficial]
        composite_sheet_into_image(final_image, final_opacity, current_sheet, current_sheet_opacity)
        
        # Update occlusion map via convolution on tiles with contributions [S3, unofficial]
        update_occlusion_map(occlusion_map, current_sheet, current_sheet_opacity)
        
        current_slab_z += slab_width_delta_s
    
    return final_image

def compute_kernel_sections(kernel_base, scaling_factor):
    # Returns a list of kernel sections with weights based on base distribution and scaling [S1, unofficial]
    sections = []
    for i in range(kernel_base.num_sections):
        weight = kernel_base.base_distribution(i) * scaling_factor
        sections.append(KernelSection(weight=weight, footprint=kernel_base.footprint))
    return sections

def transform_to_screen(voxel_center, view_params):
    # Standard perspective projection to screen coordinates [G]
    x = view_params.proj_matrix[0][0] * voxel_center.x + view_params.proj_matrix[0][3]
    y = view_params.proj_matrix[1][1] * voxel_center.y + view_params.proj_matrix[1][3]
    return Point(x, y)

def is_visible(screen_pos, occlusion_map):
    # Checks if screen position exceeds occlusion threshold [S3, unofficial]
    h, w = len(occlusion_map), len(occlusion_map[0])
    y, x = int(screen_pos.y), int(screen_pos.x)
    if 0 <= y < h and 0 <= x < w:
        return occlusion_map[y][x] < 0.95
    return True

def clip_kernel_to_slab(kernel, slab):
    # Returns kernel scaled/weighted for slab intersection [S3, unofficial]
    return Kernel(weight=kernel.weight * slab.intersection_ratio(kernel), footprint=kernel.footprint)

def rasterize_footprint(sheet, sheet_opacity, kernel, screen_pos):
    # Additive accumulation within the sheet buffer [S2, unofficial]
    h, w = len(sheet), len(sheet[0])
    for dy, dx in kernel.footprint_offsets:
        y, x = int(screen_pos.y + dy), int(screen_pos.x + dx)
        if 0 <= y < h and 0 <= x < w:
            sheet[y][x][0] += kernel.weight * kernel.color[0]
            sheet[y][x][1] += kernel.weight * kernel.color[1]
            sheet[y][x][2] += kernel.weight * kernel.color[2]
            sheet_opacity[y][x][0] += kernel.weight

def composite_sheet_into_image(final_image, final_opacity, current_sheet, current_sheet_opacity):
    # Standard back-to-front alpha compositing between sheets [G]
    h, w = len(final_image), len(final_image[0])
    for y in range(h):
        for x in range(w):
            oc = current_sheet_opacity[y][x][0]
            if oc <= 0.0:
                continue
            fc = current_sheet[y][x]
            fo = final_opacity[y][x][0]
            alpha = oc * (1.0 - fo)
            final_image[y][x][0] += fc[0] * alpha
            final_image[y][x][1] += fc[1] * alpha
            final_image[y][x][2] += fc[2] * alpha
            final_opacity[y][x][0] += alpha

def update_occlusion_map(occlusion_map, current_sheet, current_sheet_opacity):
    # Convolve sheet opacity into occlusion map, limited to tiles with contributions [S3, unofficial]
    h, w = len(occlusion_map), len(occlusion_map[0])
    for y in range(h):
        for x in range(w):
            if current_sheet_opacity[y][x][0] > 0.0:
                occlusion_map[y][x] = min(1.0, occlusion_map[y][x] + current_sheet_opacity[y][x][0])
```

**Parameters:**
*   $\Delta s$: Slab width. Typical values are not explicitly given, but it must be chosen to balance between aliasing reduction and computational cost [S3, unofficial].
*   Kernel Base Distribution: Defines the shape of the splat kernel (e.g., Gaussian) [S1, unofficial][S3, unofficial].
*   Scaling Factor: Used to adjust kernel radius based on cutting plane distance [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The integration over $\Delta s$ assumes smooth variation within the slab. If $\Delta s$ is too large, high-frequency details may be lost due to low-pass filtering [S3, unofficial].
*   **Classification:** If data values (density) are splatted before classification, the original density is smoothed and accumulated, making classification an unsolved problem that aggravates the partial volume effect [S3, unofficial].
*   **Occlusion Map Cost:** Early splat elimination requires transforming invisible voxels to screen space to determine occlusion, saving only the rasterization cost [S3, unofficial].
*   **Viewing Direction Switch:** Aligning the compositing sheet with the image plane eliminates the popping artifacts caused by sudden switches to orthogonal slice stacks in axis-aligned methods [S2, unofficial][S4, book].

## Operating characteristics
*   **Time Complexity:** Higher than axis-aligned sheet-buffered splatting because each voxel may be considered more than once (multiple splatting) as the slab advances [S2, unofficial][S4, book].
*   **Memory Complexity:** Requires storage for sheet buffers and a dynamically updated occlusion map [S2, unofficial][S3, unofficial].
*   **Performance:** Slower than raycasting for datasets with a low number of non-contributing samples. Faster than raycasting for datasets with a small number of relevant voxels and sheet buffers [S5].
*   **Strengths:** Eliminates color bleeding and popping artifacts; provides accurate reconstruction of the voxel field; produces smoother images with better anti-aliasing due to z-averaged kernels and Gaussian filters [S2, unofficial][S4, book][S5].
*   **Limitations:** More expensive due to multiple splatting of voxels; early splat elimination still requires transformation of occluded voxels; classification is problematic if density is splatted first [S2, unofficial][S3, unofficial].
*   **When to Use:** Best suited for post-classified rendering where accurate density/gradient reconstruction is needed; beneficial for rotational angiography visualization [S2, unofficial][S4, book].
*   **When Not to Use:** Not ideal for datasets with many non-contributing samples where raycasting would be faster; avoid if pre-classification is required without density smoothing issues [S3, unofficial][S4, book].

## Background
Image-aligned splatting was introduced by Mueller [S3, unofficial] (referred to as Mueller 76) to address the drawbacks of earlier axis-aligned sheet-buffered approaches, which suffered from severe brightness variations and did not allow variable DVRI interval distances [S3, unofficial]. It builds on the concept of sheet-buffered compositing but aligns the compositing sheet with the image plane to ensure consistent visual quality regardless of viewing angle [S2, unofficial]. The method utilizes analytical pre-integration of kernel slices and fast footprint rasterization for efficient projection [S3, unofficial].

## Sources
[S1] Splatting multiresolution volume data using the feature graph, http://hdl.handle.net/2117/86461, fetched 2026-10-26 (unofficial)
[S2] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S3] Volume Visualization and Volume Rendering Techniques, https://www.cs.unh.edu/~cs880/visualization/Meissner-VolRenderingEGTutorial-rdb.pdf, fetched 2026-10-27 (unofficial)
[S4] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
[S5] A Practical Evaluation of Popular Volume Rendering ..., https://courses.cs.duke.edu/spring03/cps296.8/papers/PracticalEvaluationofVolumeRenderingAlgorithms.pdf, fetched 2026-10-29
