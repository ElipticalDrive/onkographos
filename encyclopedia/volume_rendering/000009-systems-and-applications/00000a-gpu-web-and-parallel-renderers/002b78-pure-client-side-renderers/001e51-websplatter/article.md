# WebSplatter


## Overview
WebSplatter is a pure client-side WebGPU renderer for 3D Gaussian Splatting (3DGS) designed to achieve high-performance, cross-device compatibility in web browsers [S1, unofficial]. It solves the problem of inconsistent performance and stability in existing web-based 3DGS viewers by replacing CPU-based or spin-wait-based sorting with a fully GPU-based, wait-free radix sort [S1, unofficial]. Unlike its sibling [ClearView (Unified) Renderer](2460-clearview-unified-renderer.md), which focuses on unified rendering pipelines, WebSplatter specifically targets the constraints of the WebGPU API and heterogeneous hardware (from high-end desktops to mobile phones), ensuring that the system never becomes sort-bound and maintains visual fidelity comparable to native CUDA implementations [S1, unofficial].

## How it works
WebSplatter processes 3D Gaussian Splatting data through three main stages: pre-processing, sorting, and rendering. The method relies on WebGPU compute shaders for data transformation and sorting, and fragment shaders for rasterization.

### Inputs and Outputs
*   **Input**: A 3DGS scene file containing Gaussian primitives (position, covariance/scale, rotation, opacity, spherical harmonics coefficients) and camera parameters (view and projection matrices).
*   **Output**: A 2D color buffer (framebuffer) representing the rendered image.

### Data Structures
The implementation uses GPU buffers to store intermediate states. While exact byte layouts are not fully specified in the source, the following structures are implied by the WebGPU context and 3DGS requirements:
1.  **Gaussian Attributes Buffer**: Stores raw Gaussian data. Fields typically include:
    *   `position`: `vec3<f32>` (3D coordinates).
    *   `scale`: `vec3<f32>` (scaling factors).
    *   `rotation`: `vec4<f32>` (quaternion).
    *   `opacity`: `f32`.
    *   `sh_coeffs`: Array of `f32` (Spherical Harmonics).
2.  **Splat List Buffer**: Stores the projected 2D splats. Fields typically include:
    *   `center`: `vec2<f32>` (projected screen coordinates).
    *   `width`/`height`: `f32` (extent of the 2D Gaussian).
    *   `depth`: `f32` (for sorting).
    *   `original_index`: `u32` (to retrieve attributes).
3.  **Sorting Buffers**: Double-buffered arrays of indices or keys used by the radix sort algorithm.

### Numbered Steps

1.  **Pre-processing (Projection)**:
    *   For each Gaussian primitive, compute the 3D-to-2D projection using the camera's view and projection matrices.
    *   Calculate the 2D covariance matrix in screen space.
    *   Determine the bounding box (width and height) of the 2D Gaussian ellipse.
    *   Discard Gaussians that are outside the view frustum or have negligible contribution.
    *   Store the projected splat data (center, width, height, depth, original index) in the Splat List Buffer.

2.  **Sorting (Wait-Free Radix Sort)**:
    *   Sort the Splat List Buffer by depth (z-coordinate) in front-to-back order.
    *   Use a GPU-based radix sort algorithm. The source emphasizes a "wait-free" design to avoid spin-waiting, which is critical for Apple M-series chips where workgroup execution order is not guaranteed [S1, unofficial].
    *   The sort is performed entirely on the GPU using compute shaders, avoiding CPU-GPU synchronization stalls.

3.  **Rendering (Rasterization)**:
    *   Iterate through the sorted splats.
    *   For each splat, rasterize the 2D Gaussian ellipse onto the screen.
    *   Perform alpha blending:
        $$ C_{out} = C_{src} \cdot \alpha_{src} + C_{dst} \cdot (1 - \alpha_{src}) $$
        Where $C_{src}$ is the color of the current splat, $\alpha_{src}$ is its opacity, and $C_{dst}$ is the current color in the framebuffer.
    *   Update the depth buffer if necessary (though standard 3DGS often uses depth peeling or simple alpha blending without strict depth testing for the final image).

### Pseudocode
The following pseudocode outlines the core logic, focusing on the sorting and rendering pipeline. Note that WebGPU-specific shader code is abstracted into function calls.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# WebSplatter Pipeline

def render_scene(gaussians, camera):
    # Step 1: Pre-processing
    # Project 3D Gaussians to 2D screen space
    splats = []
    for g in gaussians:
        # Project position
        pos_2d = project_3d_to_2d(g.position, camera.view_matrix, camera.projection_matrix)
        
        # Calculate 2D covariance and bounding box
        cov_2d = compute_screen_covariance(g.covariance, camera.view_matrix)
        width, height = compute_bounding_box(cov_2d)
        
        # Frustum culling
        if is_in_frustum(pos_2d, width, height, camera.viewport):
            splats.append({
                'center': pos_2d,
                'width': width,
                'height': height,
                'depth': g.position.z, # Or depth in view space
                'index': g.id,
                'opacity': g.opacity,
                'color': g.color
            })
    
    # Step 2: Sorting
    # Use GPU-based wait-free radix sort
    # Sort splats by depth (ascending, front-to-back)
    sorted_indices = gpu_radix_sort(splats, key='depth')
    
    # Step 3: Rendering
    # Clear framebuffer
    clear_framebuffer()
    
    # Rasterize sorted splats
    for idx in sorted_indices:
        splat = splats[idx]
        
        # Rasterize the 2D Gaussian
        # This is typically done via fragment shader with a full-screen quad
        # or by drawing a triangle/quad for each splat.
        # For performance, instanced rendering or a single draw call with
        # a large vertex buffer is used.
        
        # Alpha blending is handled in the fragment shader
        draw_splat(splat.center, splat.width, splat.height, 
                   splat.opacity, splat.color)
    
    return get_framebuffer()

def gpu_radix_sort(data, key):
    """
    Implements a wait-free radix sort on the GPU.
    Avoids spin-waiting by using atomic operations or lock-free queues
    suitable for WebGPU compute shaders.
    """
    # 1. Extract keys (depth)
    keys = [d[key] for d in data]
    
    # 2. Perform radix sort on GPU
    # The source specifies this is >24x faster than CPU sort
    # and >4.5x faster than spin-wait sorts on M1
    sorted_data = wait_free_radix_sort_on_gpu(keys, data)
    
    return sorted_data
```

### Parameters and Typical Values
*   **Radix Sort Bins**: Not specified in the source, but standard radix sort uses 8-bit or 16-bit bins.
*   **Splat Resolution**: Not specified, but depends on the Gaussian scale.
*   **Frame Time Targets**: Real-time (>30 FPS, i.e., <33.3 ms/frame) is the goal. On high-end GPUs, frame times are ~9.5 ms [S1, unofficial].

### Edge Cases and Pitfalls
*   **Apple M1/M-Series Chips**: Spin-wait-based sorting performs poorly due to lack of workgroup execution order guarantees, leading to busy-waiting [S1, unofficial]. WebSplatter's wait-free design avoids this.
*   **Memory Constraints**: Low-end devices (e.g., Oppo Find X2, iPhone 15 Pro Max) may crash if memory requirements exceed hardware limits. WebSplatter's efficient memory usage allows it to render on these devices where baselines fail [S1, unofficial].
*   **Firefox WebGPU Instability**: Firefox may block WebGPU or report unreliable timestamp queries. WebSplatter is tested on Chrome/Edge where WebGPU is stable [S1, unofficial].
*   **Pre-processing Bottleneck**: On low-end GPUs (e.g., Nvidia MX350), pre-processing (scanning raw splats) becomes the bottleneck due to limited memory throughput [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: The sorting step is $O(N \log N)$ or $O(N)$ depending on the radix sort implementation, but the key metric is that it is GPU-accelerated and never becomes the bottleneck. Pre-processing is $O(N)$. Rendering is $O(S)$ where $S$ is the number of splats.
*   **Memory Complexity**: $O(N)$ for storing Gaussian data and intermediate splat lists. WebSplatter is optimized to fit within the memory constraints of mobile devices [S1, unofficial].
*   **Measured Performance**:
    *   **Nvidia RTX 3070**: Median frame time of 9.497 ms on the garden scene (5.8M splats), a 1.52× speedup over the fastest competing WebGPU implementation [S1, unofficial].
    *   **MacBook Air M4**: 1.06× faster than baseline. P99 latency of 116.851 ms with a standard deviation of 22.303 ms, indicating low jitter [S1, unofficial].
    *   **Redmi K70 Pro**: 1.18× faster than baseline. Renders 2.5M splats in 52.7 ms without crashing [S1, unofficial].
    *   **Intel NUC iGPU**: 2.26× faster than baseline [S1, unofficial].
    *   **Scalability**: On Nvidia MX350, scaling from 0.3M to 5.8M splats increases frame time by ~143 ms, compared to >280 ms for KeKsBoTer/web-splat (2.1× speedup) [S1, unofficial].
*   **Bottleneck Analysis**:
    *   **High-end GPUs** (RTX 3070, M-series, Redmi K70 Pro): Render-bound. Rasterization consumes 51.8% to 62.6% of frame time [S1, unofficial].
    *   **Low-end GPUs** (MX350, Intel NUC iGPU): Pre-process-bound. Pre-processing consumes up to 47.2% of frame time due to limited memory throughput [S1, unofficial].
    *   **Sort-bound**: Never occurs across all tested hardware [S1, unofficial].
*   **Strengths**:
    *   Cross-device compatibility (works on mobile, desktop, integrated graphics).
    *   Wait-free sorting avoids hardware-specific pitfalls (e.g., Apple M1).
    *   Low variance in frame time, reducing micro-stuttering.
    *   Visual fidelity comparable to native CUDA implementation [S1, unofficial].
*   **Limitations**:
    *   Dependent on WebGPU support in browsers (Firefox has stability issues).
    *   Pre-processing can be a bottleneck on very low-end hardware.

## Background
WebSplatter was introduced in the paper "WebSplatter: Enabling Cross-Device Efficient Gaussian Splatting in Web Browsers via WebGPU" [S1, unofficial]. It addresses the limitations of previous web-based 3DGS renderers, which suffered from poor performance on certain hardware (especially Apple M-series) and instability on mobile devices. The method leverages the WebGPU API to provide a high-performance, portable solution for rendering 3DGS in web browsers.

## Sources
[S1] WebSplatter: Enabling Cross-Device Efficient Gaussian Splatting in Web Browsers via WebGPU, https://doi.org/10.48550/arxiv.2602.03207, fetched 2026-10-26 (unofficial)
