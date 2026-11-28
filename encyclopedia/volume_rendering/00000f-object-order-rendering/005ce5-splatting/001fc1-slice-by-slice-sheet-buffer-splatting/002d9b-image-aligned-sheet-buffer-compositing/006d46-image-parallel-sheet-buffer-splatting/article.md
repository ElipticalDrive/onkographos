# Image-Parallel Sheet Buffer Splatting

Also called: New sheet buffer-based splatting method, Slicing slab method, new image-parallel sheet-buffer method, slicing-slab splatting

## Overview
Image-Parallel Sheet Buffer Splatting is a volume rendering algorithm that eliminates color popping artifacts inherent in traditional sheet-buffer splatting by maintaining sheet buffers parallel to the image plane regardless of camera orientation [S1]. Unlike its sibling [Image-aligned Splatting](13640-image-aligned-splatting.md), which aligns splats with image rows, this method accumulates pre-integrated kernel sections (slabs) within fixed-width slicing slabs, approximating the discrete volume rendering integral more closely by compositing kernel parts along the viewing axis rather than adding entire voxel contributions at once [S1].

## How it works
**Inputs:**
*   Volume dataset with voxel values and opacities.
*   Camera position and orientation.
*   Splatting kernel (e.g., Gaussian).
*   Slab width parameter $w_{slab}$ (typically 1.0 voxel units) [S1].

**Outputs:**
*   Final composited image.

**Data Structures:**
1.  **Pre-integrated Kernel Sections Array:** An array of 128 overlapping, pre-integrated kernel sections [S1].
    *   Spacing $\Delta s = (\text{kernel-Extent} + w_{slab}) / 128$ [S1].
    *   Each section represents a slab of thickness $w_{slab}$ integrated against the kernel function $f(r) = 0.446 \cdot e^{-2.0 \cdot r^2}$ [S1].
    *   The overlapping kernels sum to 1.0 in a unity volume [S1].
2.  **Sheet Buffer:** A 2D buffer parallel to the image plane, accumulating color and opacity contributions [S1].
3.  **Relevant Voxel List ($L_r$):** A list of voxels intersecting the current view frustum, sorted by distance from the viewing plane [S1].

**Algorithm Steps:**

1.  **Pre-computation:**
    *   Compute the array of 128 pre-integrated kernel sections using the Gaussian kernel $f(r)$ [S1].
    *   If using thick slabs for performance, normalize the footprint tables to correct for intensity differences between adding and compositing operations. For a thick slab composed of two thin slabs with section integrals $h_f$ (front) and $h_b$ (back), and voxel opacity $\alpha_v$, the normalization factor subtracted from the footprint is $2 h_b h_f + 0.5 h_b^2 h_f$ (assuming average $\alpha_v=0.5$) [S1]. This allows building slab hierarchies with thicknesses of powers of 2 [S1].

2.  **Initialization:**
    *   Define a slicing slab of fixed width $w_{slab}$ parallel to the image plane [S1].
    *   Initialize the sheet buffer to zero [G].

3.  **Voxel Processing Loop:**
    *   For each voxel in the relevant list $L_r$ (sorted by distance from the viewing plane):
        *   Determine which pre-integrated kernel sections intersect the current slicing slab [S1].
        *   Retrieve the appropriate pre-integrated kernel section(s) based on the voxel's position relative to the slab boundaries [S1].
        *   Scale the kernel section by the voxel's color $c_v$ and opacity $\alpha_v$ [S1].
        *   Add the scaled contribution to the sheet buffer [S1].
        *   *Note:* Shading is performed once per voxel, prior to or during this step, and does not repeat for each slab section [S1].

4.  **Compositing:**
    *   Once all voxels contributing to the current slicing slab have been processed, composite the accumulated sheet buffer into the cache image using front-to-back compositing [S1].
    *   The compositing of two thin slabs with integrals $h_f$ and $h_b$ and voxel opacity $\alpha_v$ results in color:
        $$c_{comp} = \alpha_v c_v (h_b^2 - h_b^2 \alpha_v h_f + h_f^2)$$
        [S1]
    *   Advance the slicing slab forward along the viewing ray [S1].

5.  **Hardware Acceleration:**
    *   Use 2D texture mapping hardware to project the footprint mappings and perform the compositing into the sheet buffer [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def image_parallel_sheet_buffer_splatting(volume, camera, kernel_extent, slab_width=1.0):
    # 1. Pre-compute kernel sections
    num_sections = 128
    delta_s = (kernel_extent + slab_width) / num_sections
    kernel_sections = precompute_kernel_sections(num_sections, delta_s, slab_width)
    
    # 2. Initialize structures
    sheet_buffer = initialize_sheet_buffer(camera.resolution)
    cache_image = initialize_cache_image(camera.resolution)
    relevant_voxels = get_relevant_voxels(volume, camera)
    sort_by_distance(relevant_voxels, camera.viewing_plane)
    
    # 3. Define slicing slab range
    # The slab moves along the viewing ray. 
    # In a full implementation, this loop iterates through depth.
    # Here we assume a single pass or simplified depth iteration for clarity.
    slab_start = camera.near_plane
    slab_end = slab_start + slab_width
    
    while slab_end <= camera.far_plane:
        clear_sheet_buffer(sheet_buffer)
        
        for voxel in relevant_voxels:
            # Determine intersection of voxel kernel with current slab
            # This involves checking if the voxel's splat footprint overlaps the slab
            # and selecting the appropriate pre-integrated section
            
            # Calculate relative position of voxel within the slab context
            # to index into kernel_sections
            section_index = get_kernel_section_index(voxel, slab_start, slab_end, delta_s)
            
            if section_index is valid:
                section = kernel_sections[section_index]
                
                # Apply normalization if using thick slabs/hierarchies
                # Normalize factor: 2*h_b*h_f + 0.5*h_b^2*h_f
                # This is pre-computed and stored in the section or applied here
                
                # Scale by voxel properties
                contribution = voxel.color * voxel.opacity * section
                
                # Add to sheet buffer using 2D texture mapping hardware logic
                # In software simulation:
                add_to_sheet_buffer(sheet_buffer, voxel.screen_position, contribution)
        
        # Composite sheet buffer to cache image
        composite_front_to_back(cache_image, sheet_buffer)
        
        # Advance slab
        slab_start += slab_width
        slab_end += slab_width

    return cache_image

def precompute_kernel_sections(num_sections, delta_s, slab_width):
    sections = []
    for i in range(num_sections):
        # Compute integral of Gaussian kernel over the slab interval
        # f(r) = 0.446 * exp(-2.0 * r^2)
        # The integration is done radially or along the slab thickness
        # depending on the specific projection geometry.
        # The source states these are "pre-integrated kernel sections".
        integral = integrate_gaussian_slab(i * delta_s, (i + 1) * delta_s, slab_width)
        sections.append(integral)
    return sections
```

**Edge Cases and Pitfalls:**
*   **Slab Width Selection:** A slab width of 2.0 produces strong periodic artifacts because slice planes section kernels differently depending on their location on the cube face, causing bright regions where thick kernel sections are composited close to the image plane [S1]. A width of 0.5 shows minor improvements but increases cost [S1]. A width of 1.0 is recommended [S1].
*   **Intensity Normalization:** Thinner slabs produce darker images due to the inequivalence of adding and compositing operations. Normalization via footprint manipulation is required to maintain similar color intensity across different slab widths [S1].
*   **Performance:** Increasing the number of sheet buffers (or reducing slab width) increases the number of compositing operations proportionally to the number of footprint mappings per voxel [S1].

## Operating characteristics
*   **Time Complexity:** The run time is dominated by polygon transform and rasterization operations. The added complexity relative to the traditional sheet-buffer method is determined by the factor $\text{splatExtent} / \text{slabWidth}$, which affects largely only the projection operations [S1].
*   **Memory Complexity:** Requires storage for the sheet buffer, cache image, and the array of 128 pre-integrated kernel sections [S1].
*   **Measured Performance:** On an SGI Onyx with InfiniteReality engine:
    *   MRI-brain dataset (1.2M relevant voxels): 16.6s for the new method vs 4.2s for the traditional method (sorting took an extra 7s) [S1].
    *   Nerve dataset (0.9M relevant voxels): 2.8s for the new method vs 9.0s for the traditional method [S1].
*   **Strengths:** Eliminates popping artifacts; reduces color bleeding; provides smoother surface appearance; allows control of accuracy via slab width [S1].
*   **Limitations:** Slower than traditional sheet-buffer splatting for some datasets due to increased compositing operations; requires careful normalization for different slab widths [S1].
*   **When to Use:** When visual quality (absence of popping) is critical and hardware acceleration for texture mapping/compositing is available [S1].
*   **When Not to Use:** When maximum rendering speed is required and popping artifacts are acceptable or can be mitigated by other means [S1].

## Background
*   **Origin:** The method was introduced to address color popping artifacts in sheet-buffer-based splatting, which occur when the orientation of compositing sheets changes suddenly as the image screen becomes more parallel to another volume face [S1].
*   **Development:** It builds on the discretized volume rendering model, approximating the integral by slicing the volume into parallel images (slabs) and compositing them [S1]. It differs from Laur and Hanrahan’s scheme of increasing the Gaussian exponent by using footprint table manipulation [S1].
*   **Usage:** Used in animated viewing of medical datasets (MRI-brain, nerve cells) where smooth illumination changes are critical [S1].

## Subtopics

### Image-Aligned Sheet-Buffer Compositing
The parent method that this technique refines. It uses sheet buffers aligned with volume axes, leading to popping artifacts. [Image-Aligned Sheet-Buffer Compositing](5594-image-aligned-sheet-buffer-compositing.md).

### Image-aligned Splatting
A sibling method that aligns splats with image rows. It differs in how it handles the sheet buffer orientation and splat projection. [Image-aligned Splatting](13640-image-aligned-splatting.md).

## Sources
[S1] Eliminating Popping Artifacts in Sheet Buffer-Based Splatting, https://www3.cs.stonybrook.edu/~mueller/papers/pop.pdf, fetched 2026-10-29
