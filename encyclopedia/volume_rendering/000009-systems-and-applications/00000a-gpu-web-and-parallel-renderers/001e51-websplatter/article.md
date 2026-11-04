# WebSplatter

Also called: WebSplatter: Enabling Cross-Device Efficient Gaussian Splatting in Web Browsers via WebGPU

## Overview
WebSplatter is a WebGPU-based renderer for 3D Gaussian Splatting (3DGS) designed to achieve high-performance, stable, and cross-device compatibility for web-based applications [S1, unofficial]. It solves the critical bottlenecks of previous web-based 3DGS implementations, specifically addressing the instability of Firefox WebGPU, excessive memory consumption on mobile devices, and severe performance degradation due to spin-wait-based sorting on Apple Silicon architectures [S1, unofficial]. WebSplatter distinguishes itself from its siblings by employing a wait-free GPU-based radix sort and a highly optimized pre-processing pipeline that prevents the renderer from becoming sort-bound, ensuring consistent frame times and low variance across a wide spectrum of hardware, from high-end desktops to low-end integrated graphics and mobile phones [S1, unofficial].

## How it works
WebSplatter implements a complete 3DGS rendering pipeline within the WebGPU API. The method consists of three primary stages: pre-processing, sorting, and rasterization.

### Inputs and Outputs
*   **Inputs:** 3D Gaussian parameters (position, covariance/scale, rotation, opacity, spherical harmonics coefficients) and camera parameters (view and projection matrices).
*   **Outputs:** A 2D color image (framebuffer) representing the rendered scene.

### Data Structures
The system processes raw splat data. While specific memory layouts for the Gaussian attributes (position, scale, rotation, opacity, SH coefficients) follow the standard 3DGS format, WebSplatter optimizes the handling of these structures during the pre-processing stage to minimize memory bandwidth usage on low-end devices [S1, unofficial].

### Steps

1.  **Pre-processing:**
    *   Transform 3D Gaussian centers into clip space using the view and projection matrices.
    *   Project the 3D covariance matrix into 2D screen space to compute the 2D bounding box and screen-space covariance.
    *   Discard Gaussians that are outside the view frustum or have negligible screen-space extent.
    *   Compute the 2D screen-space center and the 2D covariance matrix determinant for depth sorting.
    *   *Optimization:* This stage is optimized to reduce memory throughput bottlenecks on low-end GPUs by efficiently scanning raw splats [S1, unofficial].

2.  **Sorting (Wait-Free GPU Radix Sort):**
    *   Sort the visible Gaussians by their projected depth (z-coordinate) to ensure correct alpha blending.
    *   WebSplatter uses a GPU-based radix sort algorithm that is **wait-free**. This design avoids spin-waits, which are problematic on architectures like Apple M-series that lack guarantees for workgroup execution order [S1, unofficial].
    *   The sort key is the depth of the Gaussian center in clip space.

3.  **Rasterization (Rendering):**
    *   Iterate through the sorted Gaussians.
    *   For each Gaussian, compute its 2D screen-space ellipse.
    *   Rasterize the ellipse, computing the contribution to the framebuffer pixels based on the Gaussian's opacity, color (from SH coefficients), and 2D covariance.
    *   Perform alpha blending to composite the final color.

### Pseudocode
The following pseudocode outlines the core logic of the WebSplatter pipeline, focusing on the sorting and rendering stages. Note that WebGPU-specific buffer management and shader compilation are abstracted into standard compute and render pipeline calls.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def web_splatter_pipeline(gaussians, camera, width, height):
    """
    Main rendering pipeline for WebSplatter.
    
    Args:
        gaussians: List of 3D Gaussian objects with attributes:
                   - center: vec3
                   - scale: vec3
                   - rotation: quat
                   - opacity: float
                   - sh_coeffs: array (for color)
        camera: Object with view_matrix (mat4) and projection_matrix (mat4)
        width: int, framebuffer width
        height: int, framebuffer height
        
    Returns:
        ImageBuffer: Rendered 2D image
    """
    
    # Step 1: Pre-processing
    # Transform and project Gaussians to screen space
    visible_gaussians = []
    for g in gaussians:
        # Transform center to clip space
        clip_center = camera.projection_matrix @ (camera.view_matrix @ vec4(g.center, 1.0))
        
        # Check frustum culling (simplified)
        if not is_in_frustum(clip_center):
            continue
            
        # Project covariance to 2D
        # Compute 2D bounding box and screen-space covariance
        # This step is optimized in WebSplatter to reduce memory bandwidth usage
        projected_g = project_gaussian(g, camera)
        
        if projected_g.is_valid:
            visible_gaussians.append(projected_g)
            
    # Step 2: Sorting
    # WebSplatter uses a wait-free GPU-based radix sort
    # Sort by depth (z-coordinate in clip space)
    # This avoids spin-waits on Apple Silicon and other architectures
    sorted_gaussians = wait_free_radix_sort(visible_gaussians, key=lambda g: g.clip_center.z)
    
    # Step 3: Rasterization
    # Initialize framebuffer
    framebuffer = ImageBuffer(width, height, format='RGBA')
    
    # Render sorted Gaussians
    for g in sorted_gaussians:
        # Compute 2D screen-space ellipse parameters
        # Compute contribution to each pixel within the bounding box
        render_gaussian_ellipse(g, framebuffer, width, height)
        
    return framebuffer

def wait_free_radix_sort(data, key):
    """
    Wait-free GPU-based radix sort implementation.
    Uses atomic operations and barrier synchronization that does not rely on 
    workgroup execution order guarantees, making it safe for Apple M-series chips.
    """
    # Implementation details of the radix sort are handled by the WebGPU compute shader.
    # The key aspect is the absence of spin-waits.
    pass

def render_gaussian_ellipse(g, framebuffer, width, height):
    """
    Rasterize a single 2D Gaussian ellipse.
    """
    # Compute bounding box of the ellipse in screen space
    bbox = compute_2d_bounding_box(g)
    
    # Iterate over pixels in the bounding box
    for y in range(bbox.min_y, bbox.max_y):
        for x in range(bbox.min_x, bbox.max_x):
            if is_inside_ellipse(x, y, g):
                # Compute alpha and color contribution
                alpha, color = compute_gaussian_contribution(x, y, g)
                
                # Alpha blend into framebuffer
                framebuffer.blend_pixel(x, y, color, alpha)
```

### Parameters and Typical Values
*   **Sort Algorithm:** GPU-based radix sort.
*   **WebGPU API:** Used for all GPU operations.
*   **Scene Complexity:** Tested with scenes ranging from 0.3M splats (van gogh) to 5.8M splats (garden) [S1, unofficial].

### Edge Cases and Pitfalls
*   **Apple Silicon Sorting:** Spin-wait-based sorting (e.g., KeKsBoTer/web-splat) performs poorly on Apple M1 chips due to the lack of guarantees for workgroup execution order, leading to significant busy-waiting. WebSplatter's wait-free design avoids this [S1, unofficial].
*   **Memory Constraints:** Baselines like KeKsBoTer/web-splat crash on devices with limited memory (e.g., Oppo Find X2) due to excessive memory requirements. WebSplatter's optimized pre-processing and sorting reduce memory pressure, allowing it to render on all tested devices [S1, unofficial].
*   **Firefox WebGPU Instability:** Firefox WebGPU is unstable across tested devices, with crashes and blocklists. WebSplatter is tested on Google Chrome, which provides a more stable WebGPU implementation [S1, unofficial].
*   **Timestamp Query Discrepancies:** On Intel NUC, Firefox WebGPU timestamp queries are unreliable (built-in queries: 2.8 ms vs JavaScript API: 258.36 ms). WebSplatter's performance measurements are robust against such API inconsistencies [S1, unofficial].

## Operating characteristics
### Time and Memory Complexity
*   **Time Complexity:** The system is never sort-bound across all tested hardware. On high-end GPUs (RTX 3070, Apple M-series, Redmi K70 Pro), the system is render-bound, with rasterization consuming 51.8% to 62.6% of frame time [S1, unofficial]. On low-end GPUs (Nvidia MX350, Intel NUC iGPU), the system is pre-process-bound, consuming up to 47.2% of frame time due to limited memory throughput scanning raw splats [S1, unofficial].
*   **Memory Complexity:** WebSplatter's optimized pre-processing and wait-free sorting reduce memory pressure, allowing it to run on devices with limited VRAM where other baselines crash [S1, unofficial].

### Measured Performance
*   **Nvidia RTX 3070:** Median total frame time of 9.497 ms on the garden scene, a 1.52× speedup over the fastest competing WebGPU implementation [S1, unofficial].
*   **MacBook Air M4:** 1.06× faster than baseline. On the garden scene (5.8M splats), standard deviation of 22.303 ms and P99 latency of 116.851 ms, indicating low variance and mitigation of micro-stuttering [S1, unofficial].
*   **Redmi K70 Pro:** 1.18× faster than baseline. Renders the truck scene (2.5M splats) in 52.7 ms without crashing, while KeKsBoTer/web-splat crashes [S1, unofficial].
*   **Intel NUC (iGPU):** 2.26× faster than baseline [S1, unofficial].
*   **MacBook Pro M1:** Over 4.5× faster than KeKsBoTer/web-splat due to the avoidance of spin-wait-based sorting [S1, unofficial].
*   **Scalability:** On Nvidia MX350, scaling from van gogh (0.3M splats) to garden (5.8M splats) increases WebSplatter frame time by ~143 ms, compared to >280 ms for KeKsBoTer/web-splat, yielding a 2.1× speedup on the largest scene [S1, unofficial].

### Strengths and Limitations
*   **Strengths:**
    *   High performance across a wide range of hardware, from high-end desktops to low-end mobile devices.
    *   Wait-free sorting ensures stability and performance on Apple Silicon.
    *   Low variance in frame times, reducing micro-stuttering.
    *   Successful rendering on all tested devices, including those with limited memory.
    *   Visual fidelity comparable to the official native CUDA implementation [S1, unofficial].
*   **Limitations:**
    *   Performance is still bound by rasterization on high-end GPUs, limiting further speedups without algorithmic changes to the rendering stage.
    *   Pre-processing becomes a bottleneck on low-end GPUs, limited by memory throughput [S1, unofficial].

### When to Use
*   Use WebSplatter for web-based 3DGS applications requiring high performance and stability across diverse hardware, especially when targeting Apple Silicon devices or low-end integrated graphics.
*   Use when visual fidelity comparable to native implementations is required.

### When Not to Use
*   Do not use if the target environment relies on Firefox WebGPU, due to known instability and blocklists.
*   Do not use if the hardware has extremely limited memory and the scene complexity is very high, although WebSplatter is more memory-efficient than baselines, extreme cases may still be challenging [S1, unofficial].

## Background
WebSplatter was introduced in the paper "WebSplatter: Enabling Cross-Device Efficient Gaussian Splatting in Web Browsers via WebGPU" [S1, unofficial]. The work addresses the growing need for efficient 3D Gaussian Splatting rendering in web browsers, leveraging the WebGPU API for hardware-accelerated graphics. Previous web-based 3DGS implementations suffered from performance bottlenecks, particularly in sorting and memory management, and instability on certain hardware and browsers. WebSplatter's contributions include a wait-free GPU-based radix sort and an optimized pre-processing pipeline, enabling efficient and stable 3DGS rendering across a wide range of devices [S1, unofficial].

## Sources
[S1] WebSplatter: Enabling Cross-Device Efficient Gaussian Splatting in Web Browsers via WebGPU, https://doi.org/10.48550/arxiv.2602.03207, fetched 2026-10-26 (unofficial)
