# Multi-Resolution Rendering for Clouds
Also called: Low-resolution first rendering for clouds, Progressive resolution cloud rendering

## Overview
Multi-resolution rendering for clouds is an acceleration technique that limits the total number of volume samples by initially rendering the cloud field at a low resolution to identify regions requiring detailed sampling, then accumulating the final image with higher-resolution passes [S1, unofficial]. Unlike standard adaptive sampling which varies sample counts per pixel based on local error metrics, this method varies the spatial resolution of the entire rendering pass, using the coarse pass as a guide to skip or reduce expensive high-frequency evaluations in empty or uniform areas [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric cloud density field $\rho(x,y,z)$, a camera configuration, and a target output resolution $R_{target}$.
*   **Output:** A final rendered image $I_{final}$.

**Data Structures:**
*   **Coarse Buffer:** A framebuffer of resolution $R_{low} \ll R_{target}$ storing intermediate density or color values.
*   **Fine Buffer:** A framebuffer of resolution $R_{target}$ storing the accumulated result.

**Steps:**
1.  **Low-Resolution Pass:** Render the cloud volume at resolution $R_{low}$. This involves casting rays and performing volume integration (e.g., ray marching) at the lower resolution. This pass is computationally cheap and provides a quick indication of where the cloud density is significant [S1, unofficial].
2.  **Analysis:** Identify pixels in the low-resolution buffer that correspond to empty space or low-density regions. These areas require fewer samples in the subsequent pass [S1, unofficial].
3.  **High-Resolution Accumulation:** Render the cloud volume at resolution $R_{target}$. For pixels identified as low-complexity in step 2, reduce the number of samples or skip the ray march entirely, using the low-resolution result or a constant background value. For high-complexity pixels, perform full-resolution ray marching [S1, unofficial].
4.  **Composition:** Combine the results. The low-resolution pass acts as a guide; the high-resolution pass refines the details only where necessary. The final image is accumulated from these passes [S1, unofficial].

**Parameters:**
*   $R_{low}$: The initial low resolution. Typical values are not given in the sources.
*   $R_{target}$: The final output resolution.

**Edge Cases and Pitfalls:**
*   **Aliasing:** Switching between resolutions may introduce visible seams or aliasing artifacts if not handled carefully [S1, unofficial].
*   **Complexity:** The technique adds complexity to the rendering pipeline, as it requires managing multiple passes and buffers [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_resolution_cloud_rendering(camera, cloud_field, R_target):
    # Define low resolution, typically a fraction of target
    R_low = R_target / 4  # Example ratio, not specified in sources
    
    # Step 1: Low-resolution pass
    # Render at low resolution to get a quick indication of density
    low_res_image = render_cloud_at_resolution(camera, cloud_field, R_low)
    
    # Step 2: Identify regions needing high resolution
    # This could be based on density threshold or gradient
    # Not fully specified in sources, so we assume a simple density check
    high_detail_mask = compute_high_detail_mask(low_res_image, threshold=0.1)
    
    # Step 3: High-resolution pass with adaptive sampling
    # Initialize final image
    final_image = zeros_like(R_target)
    
    for each pixel (x, y) in R_target:
        if high_detail_mask[x, y]:
            # Full resolution ray marching for high-detail areas
            color = ray_march_cloud(camera, cloud_field, x, y, samples=64)
        else:
            # Skip or use low-res result for low-detail areas
            # Bilinear interpolation from low_res_image to match R_target
            color = interpolate(low_res_image, x, y)
        
        final_image[x, y] = color
        
    return final_image
```

## Subtopics

### [Iteratively Increasing Resolution](9448-iteratively-increasing-resolution.md)
Iteratively Increasing Resolution is a rendering strategy that reduces computation time by starting at a low resolution and gradually increasing it to the target final resolution. It solves the problem of expensive high-resolution ray-surface intersection tests by deferring them until necessary, using cone-tracing to guarantee that no surface points are skipped during the resolution transition. Unlike standard adaptive sampling which varies sample counts per pixel, this method varies the spatial resolution of the entire rendering pass, building on the acceleration principles of [Multi-Resolution Rendering for Clouds](7260-multi-resolution-rendering-for-clouds.md) but applied specifically to sphere tracing and SDF rendering.

## Operating characteristics
*   **Time Complexity:** Reduces total computation time by avoiding full-resolution ray marching in empty or uniform regions. The cost is dominated by the low-resolution pass plus the high-resolution pass only for relevant pixels.
*   **Memory Complexity:** Requires additional memory for the low-resolution buffer and the high-resolution buffer.
*   **Strengths:** Effective for scenes with large empty spaces or uniform cloud regions, significantly reducing sample count.
*   **Limitations:** Adds pipeline complexity; may introduce artifacts at resolution boundaries; not suitable for highly dynamic or temporally coherent scenes without additional techniques like temporal accumulation [S1, unofficial].
*   **When to Use:** When rendering static or slowly moving clouds with significant empty space.
*   **When Not to Use:** When high temporal coherence is required and the overhead of managing multiple passes outweighs the savings, or for highly detailed, dense clouds where most pixels require full resolution.

## Background
The technique was proposed by the author of the three.js forum post "Efficient volumetric clouds" as a potential optimization for cloud rendering [S1, unofficial]. The author considered it but did not implement it, suspecting it could be very effective. The idea is to use a low-resolution pass to guide a higher-resolution pass, limiting the number of samples needed [S1, unofficial].

## Sources
[S1] Efficient volumetric clouds - Resources - three.js forum, https://discourse.threejs.org/t/efficient-volumetric-clouds/66067, fetched 2026-10-29 (unofficial)
