# Multiresolution Rendering with Data Reduction

Also called: Adaptive Decompression Rendering, Super-linear Multiresolution Speed-up

## Overview
Multiresolution Rendering with Data Reduction is a direct volume ray casting technique that exploits hierarchical data structures to achieve super-linear performance increases by computing fewer voxels and increasing step lengths for low-resolution blocks [S1, unofficial]. It distinguishes itself from standard multiresolution methods by explicitly linking data compression ratios to rendering speed, allowing for aggressive reduction in less critical regions while maintaining interactive frame rates [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume dataset (e.g., $512^3$ voxels), a data reduction ratio, a segment length parameter, and a number of rays (LAO rays).
*   **Output:** A rendered image with reduced computational cost, potentially with increased mean error proportional to the reduction ratio.

**Data Structures:**
The method relies on a multiresolution hierarchy (e.g., wavelet or Laplacian pyramid) where data is stored at multiple resolutions. The specific bit-packing is not detailed in the sources, but the structure supports querying blocks at varying resolutions [S1, unofficial].

**Parameters:**
*   **Data Reduction Ratio:** The compression factor applied to volume blocks. Typical values observed: 8.5:1, 15.7:1, 31.4:1, 92.8:1 [S1, unofficial].
*   **Segment Length:** The number of voxels processed per step in low-resolution blocks. A typical value cited is 16 voxels [S1, unofficial].
*   **LAO Ray Count:** The number of rays used for lighting calculations (e.g., Local Ambient Occlusion). Typical values: 16, 64, 128 [S1, unofficial].

**Steps:**
1.  **Initialize Ray Casting:** For each pixel in the viewport, cast a ray through the volume.
2.  **Traverse Hierarchy:** As the ray traverses the volume, determine the appropriate resolution level for the current block based on the data reduction strategy.
3.  **Adaptive Step Length:** For low-resolution blocks, increase the step length to skip over voxels, effectively reducing the number of samples computed [S1, unofficial].
4.  **Sample and Accumulate:** Sample the volume data at the determined resolution and accumulate color and opacity along the ray.
5.  **Lighting Calculation:** Compute lighting (e.g., LAO) using the specified number of rays. Fewer rays reduce accuracy but increase speed [S1, unofficial].
6.  **Output:** Composite the final pixel color.

**Pitfalls and Edge Cases:**
*   **Shadow Volume Sensitivity:** Sensitivity to shadowing increases at lower resolutions ($32^3, 64^3, 128^3, 256^3$). Careful selection of segment length and reduction ratio is required to maintain accuracy [S1, unofficial].
*   **Error Scaling:** Mean error increases with higher data reduction ratios. Observed mean errors: 0.6 at 8.5:1, 1.2 at 15.7:1, 1.6 at 31.4:1 [S1, unofficial].
*   **Ray Count Sensitivity:** The number of LAO rays significantly impacts error. Observed mean errors: 0.3 for 128 rays, 0.7 for 64 rays, 0.9 for 16 rays [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_multiresolution_volume(volume_hierarchy, viewport_width, viewport_height, 
                                  reduction_ratio, segment_length, lao_ray_count):
    """
    Renders a volume using multiresolution data reduction.
    
    Args:
        volume_hierarchy: Hierarchical data structure supporting multi-resolution queries.
        viewport_width: Width of the output image.
        viewport_height: Height of the output image.
        reduction_ratio: Target compression factor (e.g., 8.5, 15.7).
        segment_length: Number of voxels to skip in low-res blocks (e.g., 16).
        lao_ray_count: Number of rays for lighting calculation (e.g., 128).
        
    Returns:
        Image buffer.
    """
    image = empty_buffer(viewport_width, viewport_height)
    
    for y in range(viewport_height):
        for x in range(viewport_width):
            ray = cast_ray_through_pixel(x, y, viewport_width, viewport_height)
            accumulated_color = (0, 0, 0, 0)
            current_pos = ray.origin
            
            while current_pos is inside_volume:
                # Determine resolution level based on reduction ratio and position
                # [G] Logic to map reduction_ratio to specific resolution level in hierarchy
                resolution_level = get_resolution_level(current_pos, reduction_ratio)
                
                # Get block at current resolution
                block = volume_hierarchy.get_block(current_pos, resolution_level)
                
                # Determine step size based on resolution
                # [G] Step size increases as resolution decreases
                step_size = calculate_step_size(resolution_level, segment_length)
                
                # Sample volume data
                sample = block.sample(current_pos)
                
                # Accumulate color and opacity
                accumulated_color = composite(accumulated_color, sample)
                
                # Advance ray
                current_pos = current_pos + ray.direction * step_size
                
                # Early termination if fully opaque
                if accumulated_color.a >= 1.0:
                    break
            
            # Apply lighting (e.g., LAO)
            # [G] Lighting calculation depends on lao_ray_count
            lighting = compute_lao_lighting(current_pos, volume_hierarchy, lao_ray_count)
            final_color = apply_lighting(accumulated_color, lighting)
            
            image[y, x] = final_color
            
    return image
```

## Subtopics

### ROI-based High-Resolution Rendering
A rendering strategy to handle prohibitively large datasets by first rendering the entire volume at reduced resolution, then refining a user-selected region of interest (ROI) at full resolution [S2]. The process involves: 1. Rendering the entire volume at reduced resolution. 2. User indicates a smaller ROI. 3. Producing a high-resolution volume rendering for the selected ROI [S2]. This reduces initial memory and compute requirements by deferring high-resolution sampling to a smaller sub-volume, addressing datasets exceeding 60 GB [S2]. It is designed specifically for tiled cryo-imaging datasets (e.g., 4×5 tiling, 40μm section thickness) where full-resolution rendering exceeds practical GPU memory limits [S2].

## Operating characteristics
*   **Performance:** Achieves super-linear performance increases by computing fewer voxels and increasing step lengths for low-resolution blocks [S1, unofficial].
*   **Accuracy:** Mean error increases with higher data reduction ratios (0.6 at 8.5:1, 1.2 at 15.7:1, 1.6 at 31.4:1) [S1, unofficial]. Error is also sensitive to the number of LAO rays (0.3 for 128 rays, 0.7 for 64 rays, 0.9 for 16 rays) [S1, unofficial].
*   **Memory:** Reduces memory requirements by using lower-resolution data for less critical regions [S2].
*   **Use Cases:** Suitable for interactive medical visualization and virtual autopsy applications where real-time performance on large datasets is required [S1, unofficial]. Also used for large tiled cryo-imaging datasets [S2].
*   **Limitations:** Shadow volume sensitivity increases at lower resolutions, requiring careful parameter selection [S1, unofficial].

## Background
The method was developed by Patric Ljung at Siemens Corporate Research, Princeton, USA, as part of research into advanced illumination techniques for GPU-based volume ray casting [S1, unofficial]. It was presented in 2008 [S1, unofficial]. The approach addresses the need for real-time performance in global lighting calculations for volumetric data [S1, unofficial].

## Sources
[S1] GPU-Based Volume Ray-Casting with Advanced Illumination, https://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/viscom/2008/advanced_illumination.pdf, fetched 2026-10-29 (unofficial)
[S2] Enhanced volume rendering techniques for high-resolution color cryo-imaging data, https://doi.org/10.1117/12.813756, fetched 2026-11-08
