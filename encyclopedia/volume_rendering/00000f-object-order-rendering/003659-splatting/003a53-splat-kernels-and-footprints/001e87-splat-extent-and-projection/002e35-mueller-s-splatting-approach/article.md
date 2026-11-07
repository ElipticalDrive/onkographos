# Mueller's Splatting Approach

Also called: Feed Forward Convolution, Footprint-based Splatting, Mueller splatting, Object Order Algorithm, Splatting (Feed Forward Convolution), Splatting Algorithms

## Overview
Mueller's Splatting Approach is an object-order direct volume rendering technique that treats a gray-scale volume as a 3D discrete density function, convolving each voxel with a reconstruction kernel to determine its contribution to the image plane [S1, S4, unofficial]. It improves upon basic splatting by aligning sheet buffers parallel to the image plane (image-aligned splatting), which eliminates the "popping" artifacts and brightness variations inherent in axis-aligned sheet approaches [S3, S6]. Unlike ray casting, it exploits cache coherency by sequentially traversing voxels, though it introduces low-pass filtering that blurs sharp boundaries [S1, unofficial][S6].

## How it works
**Inputs:**
- Volume data: A 3D array of density samples $f(s)$, where $s = (x_s, y_s, z_s)$ represents the spatial coordinates of a voxel [S1, S4, unofficial].
- Reconstruction kernel: A 3D function $h(x, y, z)$, typically a radially symmetric Gaussian [S3, S6].
- Transfer Function (TF): Maps density values to color $C$ and opacity $\alpha$ [S3].
- View parameters: Camera position, orientation, and projection type (orthographic or perspective) [S1, unofficial][S6].
- Slab width: $\Delta s$, the thickness of the slabs aligned parallel to the image plane [S6].

**Outputs:**
- Final 2D image with color and opacity values for each pixel [S3].

**Data Structures:**
1. **Footprint Table:** A 2D lookup table storing the analytically integrated kernel function along a traversing ray. Entries store weights for pixels relative to the projected voxel center. Resolution is much higher than the image plane to reduce aliasing [S1, S4, unofficial][S6].
2. **Sheet Buffer:** A 2D buffer storing accumulated color and opacity for a specific slab of voxels. It is aligned parallel to the image plane [S6].
3. **Occlusion Map:** A dynamically constructed map used for early splat elimination. It tracks which screen regions are already opaque enough to cull subsequent splats [S6].

**Formulas:**
The contribution of a single sample $s$ to a point $(x, y, z)$ in space is defined by [S1, S4, unofficial]:
$$C(x, y, z) = h(x - x_s, y - y_s, z - z_s) \cdot f(s)$$

To project this to the 2D image plane, we integrate along the viewing ray (axis $u$ parallel to the ray) [S1, S4, unofficial]:
$$C(x, y) = f(s) \int_{-\infty}^{\infty} h(x - x_s, y - y_s, u) \, du$$

The integral term is independent of sample density and defines the **Footprint Function** $F(x, y)$ [S1, S4, unofficial]:
$$F(x, y) = \int_{-\infty}^{\infty} h(x - x_s, y - y_s, u) \, du$$
where $(x, y)$ is the displacement from the sample's projected center.

For image-aligned splatting, voxels are grouped into slabs of width $\Delta s$. The effective color $C_\lambda(s_i)$ and opacity $\alpha(s_i)$ for a slab $i$ are computed by averaging the kernel contributions across the slab thickness [S6]:
$$\alpha(s_i) = \frac{\int_{i\Delta s}^{(i+1)\Delta s} \alpha(s) \, ds}{\Delta s}$$
$$C_\lambda(s_i) = \frac{\int_{i\Delta s}^{(i+1)\Delta s} C_\lambda(s) \, ds}{\Delta s}$$
$$f(s_i) = \frac{\int_{i\Delta s}^{(i+1)\Delta s} f(s) \, ds}{\Delta s}$$

**Parameters:**
- **Footprint Table Size:** Small tables cause blocky images; large tables smooth details and consume more memory [S1, unofficial].
- **Sampling Method:** Nearest-neighbor (fast, aliasing) or bilinear interpolation (smoother, slower) when sampling the view-transformed footprint table [S1, unofficial].
- **Reconstruction Filter:** Choices include cone, Gaussian, Sinc, or bilinear functions [S1, unofficial]. Gaussian kernels are common for their anti-aliasing properties but cause signal smoothing [S6].

**Algorithm Steps:**
1. **Pre-compute Footprint Table:** Evaluate the integral in Equation 2.22 on a high-resolution grid to generate the generic footprint table [S1, S4, unofficial].
2. **Initialize Sheet Buffers:** Create sheet buffers aligned parallel to the image plane. Initialize the occlusion map [S6].
3. **Sort Voxels into Slabs:** Group voxels into slabs of width $\Delta s$ based on their depth relative to the image plane [S6].
4. **Process Each Slab (Back-to-Front or Front-to-Back):**
    a. **Transform:** For each voxel in the slab, transform its center into screen space [S6].
    b. **Check Occlusion:** Index into the occlusion map to test if the voxel's footprint overlaps already opaque regions. If fully occluded, skip rasterization [S6].
    c. **Rasterize Splat:** If visible, center the footprint table on the projected screen location of the voxel. Sample the footprint table to determine weights for affected pixels. Multiply weights by the voxel's density $f(s)$ and transfer function outputs (color/opacity) [S1, S4, unofficial][S6].
    d. **Accumulate:** Add the weighted contributions to the current sheet buffer [S6].
    e. **Update Occlusion Map:** After compositing the sheet buffer into the final image, perform a convolution operation on the occlusion map, limited to tiles that received splat contributions, to update visibility for subsequent slats [S6].
5. **Composite:** Blend the sheet buffer with the previous accumulation using the Porter-Duff over operator (alpha blending) [S1, unofficial][S6].
6. **Repeat:** Continue until all slabs are processed.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mueller_splatting(volume, kernel, transfer_function, view_matrix, image_width, image_height, slab_width):
    # Pre-compute generic footprint table
    # F(x,y) = integral of kernel along viewing ray
    footprint_table = precompute_footprint_table(kernel, resolution=image_width * 4)
    
    # Initialize buffers
    sheet_buffer = zeros((image_width, image_height, 4)) # RGBA
    final_image = zeros((image_width, image_height, 4))
    occlusion_map = zeros((image_width, image_height)) # Track accumulated opacity
    
    # Sort voxels into slabs parallel to image plane
    slabs = group_voxels_into_slabs(volume, view_matrix, slab_width)
    
    # Process slabs in depth order (e.g., back-to-front)
    for slab in slabs:
        current_sheet = zeros((image_width, image_height, 4))
        
        for voxel in slab:
            # Transform voxel center to screen space
            screen_pos = transform_to_screen(voxel.center, view_matrix)
            
            # Check occlusion
            if is_occluded(screen_pos, footprint_table.radius, occlusion_map):
                continue
                
            # Get voxel properties
            density = volume[voxel.index]
            color, opacity = transfer_function(density)
            
            # Rasterize footprint
            # Center footprint table on screen_pos
            # Sample footprint table for each pixel in footprint extent
            for pixel_offset in footprint_table.extent:
                pixel_coord = screen_pos + pixel_offset
                if is_inside_image(pixel_coord, image_width, image_height):
                    weight = sample_footprint(footprint_table, pixel_offset)
                    contribution = weight * density * color
                    current_sheet[pixel_coord] += contribution
                    
                    # Update local opacity for occlusion map update later
                    # Note: Simple accumulation for occlusion check
                    local_opacity = weight * density * opacity
                    current_sheet[pixel_coord, 3] += local_opacity # Accumulate opacity in sheet
        
        # Composite sheet into final image
        # Use Porter-Duff over operator
        for y in range(image_height):
            for x in range(image_width):
                src = current_sheet[y, x]
                dst = final_image[y, x]
                alpha_src = src[3]
                alpha_dst = dst[3]
                
                # Blend
                final_image[y, x, :3] = src[:3] + dst[:3] * (1 - alpha_src)
                final_image[y, x, 3] = alpha_src + alpha_dst * (1 - alpha_src)
                
                # Update occlusion map with the new accumulated opacity
                occlusion_map[y, x] = final_image[y, x, 3]
                
        # Optional: Update occlusion map efficiently using convolution on tiles
        # update_occlusion_map(occlusion_map, current_sheet)

    return final_image

def is_occluded(screen_pos, radius, occlusion_map):
    # Check if all pixels in the footprint extent have opacity >= threshold
    for offset in range(-radius, radius + 1):
        for offset_y in range(-radius, radius + 1):
            x = int(screen_pos.x) + offset
            y = int(screen_pos.y) + offset_y
            if 0 <= x < occlusion_map.shape[1] and 0 <= y < occlusion_map.shape[0]:
                if occlusion_map[y, x] < 0.99: # Threshold for "opaque"
                    return False
    return True
```

## Subtopics

### [Multi-texture Mapping Splatting](7515-multi-texture-mapping-splatting.md)
A hardware-accelerated variant that replaces 3D kernel evaluation with pre-computed 2D footprint lookups and leverages texture mapping hardware for rasterization, trading increased memory usage for reduced sorting complexity.

## Operating characteristics
- **Time Complexity:** $O(V \cdot K)$, where $V$ is the number of voxels and $K$ is the number of pixels in the footprint. Cache coherency improves performance compared to ray casting's random access [S1, unofficial].
- **Memory Complexity:** $O(I \cdot S + F)$, where $I$ is image size, $S$ is number of slabs, and $F$ is footprint table size. Sheet buffers and occlusion maps require additional memory [S6].
- **Performance:** Faster than ray casting for sparse volumes due to early splat elimination. However, it processes all relevant voxels, unlike ray casting which can skip empty space more aggressively with trilinear reconstruction [S1, unofficial][S6].
- **Strengths:** Smooth surfaces without staircase artifacts; supports incremental rendering; good cache coherency; reduced aliasing with Gaussian kernels [S1, unofficial][S6].
- **Limitations:** Blurred images due to spherically symmetric kernels; "color bleeding" artifacts in basic implementations (solved by sheet buffers); "popping" artifacts if slabs are not aligned correctly (solved by image-aligned slabs); early splat elimination still requires transformation of occluded voxels [S1, unofficial][S3, S6].
- **When to Use:** When smooth rendering and cache efficiency are prioritized over sharp boundaries; suitable for hardware acceleration via texture mapping.
- **When Not to Use:** When sharp object boundaries are critical (use ray casting with trilinear reconstruction); when memory is extremely constrained (large footprint tables and sheet buffers).

## Background
Mueller's Splatting Approach was developed by Mueller et al. to improve upon Westover's original splatting algorithm [S3, S6]. Westover introduced splatting as an object-order technique using axis-aligned sheet buffers, which suffered from "popping" artifacts and brightness variations as the view changed [S3]. Mueller proposed aligning the sheet buffers parallel to the image plane (image-aligned splatting), which eliminated these artifacts and allowed for variable DVRI interval distances $\Delta s$ [S6]. This method was later accelerated with GPUs [S3]. It is a form of feed-forward convolution, spreading input energy to many output pixels [S5, unofficial].

## Sources
[S1] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S2] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S3] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S4] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
[S5] PDF Direct Volume Rendering, https://www.cs.unh.edu/~cs880/lectureNotes/DVR.pdf, fetched 2026-10-28 (unofficial)
[S6] A Practical Evaluation of Popular Volume Rendering ..., https://courses.cs.duke.edu/spring03/cps296.8/papers/PracticalEvaluationofVolumeRenderingAlgorithms.pdf, fetched 2026-10-29
