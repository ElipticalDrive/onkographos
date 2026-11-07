# WebSplatter

Also called: WebSplatter: Enabling Cross-Device Efficient Gaussian Splatting in Web Browsers via WebGPU

## Overview
WebSplatter is a WebGPU-based renderer for 3D Gaussian Splatting (3DGS) designed to achieve real-time performance and stability across heterogeneous hardware, ranging from high-end desktop GPUs to low-end integrated graphics and mobile devices [S1, unofficial]. It solves the critical bottlenecks of previous web-based 3DGS implementations, specifically the instability of spin-wait-based sorting on Apple Silicon and the excessive memory consumption that causes crashes on mobile browsers [S1, unofficial]. Unlike its siblings, WebSplatter employs a wait-free GPU-based radix sort and optimized pre-processing pipelines to ensure the system never becomes sort-bound and maintains visual fidelity comparable to native CUDA implementations [S1, unofficial].

## How it works
WebSplatter implements the 3DGS rendering pipeline within the WebGPU API, optimizing the three core stages: pre-processing, sorting, and rendering [S1, unofficial].

### Inputs and Outputs
*   **Inputs**: 3D Gaussian parameters (position $\mathbf{p}$, covariance matrix $\mathbf{C}$ or rotation/quaternion $\mathbf{q}$, scale $\mathbf{s}$, opacity $\alpha$, spherical harmonics coefficients $\mathbf{c}$) and camera parameters (view matrix $\mathbf{V}$, projection matrix $\mathbf{P}$) [S1, unofficial].
*   **Outputs**: A 2D color image (framebuffer) with alpha blending [S1, unofficial].

### Data Structures
The implementation relies on standard 3DGS data structures optimized for GPU memory access:
1.  **Gaussian Attributes**: Structs containing position (vec3), rotation (vec4), scale (vec3), opacity (f32), and SH coefficients (vec3 per band) [S1, unofficial].
2.  **Depth Buffer**: Used for depth sorting and occlusion culling [S1, unofficial].
3.  **Tile Boundaries**: Pre-computed tile extents for the rasterization stage [S1, unofficial].

### Numbered Steps

1.  **Pre-processing (GPU-based)**:
    *   Transform Gaussian centers from world space to camera space using the view matrix $\mathbf{V}$ [S1, unofficial].
    *   Project centers to screen space using the projection matrix $\mathbf{P}$ [S1, unofficial].
    *   Compute the 2D covariance matrix $\mathbf{T}$ for each Gaussian by linearizing the 3D covariance $\mathbf{C}$ and applying the Jacobian of the projection transformation [S1, unofficial].
    *   Calculate the 2D bounding box for each Gaussian to determine which screen tiles it overlaps [S1, unofficial].
    *   Filter out Gaussians that are outside the view frustum or have negligible contribution [S1, unofficial].
    *   *Note*: On low-end GPUs, this stage scans raw splats and is memory-throughput bound [S1, unofficial].

2.  **Sorting (Wait-free GPU Radix Sort)**:
    *   Sort Gaussians by tile ID, then by depth within each tile [S1, unofficial].
    *   WebSplatter uses a wait-free radix sort algorithm [S1, unofficial].
    *   This avoids the spin-waiting issues found in other implementations (e.g., KeKsBoTer/web-splat) on Apple M-series chips, where workgroup execution order is not guaranteed [S1, unofficial].
    *   The sort is performed entirely on the GPU using compute shaders [S1, unofficial].

3.  **Rendering (Rasterization)**:
    *   Rasterize the sorted Gaussians as screen-space quads [S1, unofficial].
    *   For each pixel, accumulate color and opacity from overlapping Gaussians using alpha blending [S1, unofficial].
    *   The rasterization stage is the primary bottleneck on high-end GPUs, consuming 51.8% to 62.6% of frame time [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# WebSplatter Pipeline Implementation
# Assumes WebGPU Compute and Render pipelines are initialized

def web_splatter_pipeline(gaussians, camera, screen_width, screen_height):
    """
    Main rendering loop for WebSplatter.
    
    Args:
        gaussians: List of Gaussian objects with attributes (pos, rot, scale, opacity, sh)
        camera: Camera object with view_matrix and projection_matrix
        screen_width: int
        screen_height: int
        
    Returns:
        ImageData: The rendered frame
    """
    
    # Step 1: Pre-processing (Compute Shader)
    # Transforms Gaussians to camera space and projects to screen space
    # Computes 2D covariance and tile assignments
    projected_gaussians = gpu_compute_preprocess(
        gaussians, 
        camera.view_matrix, 
        camera.projection_matrix,
        screen_width, 
        screen_height
    )
    
    # Step 2: Sorting (Compute Shader - Wait-free Radix Sort)
    # Sorts by tile ID, then by depth
    # This step is optimized to avoid spin-waits on Apple Silicon
    sorted_gaussians = gpu_compute_radix_sort_wait_free(projected_gaussians)
    
    # Step 3: Rendering (Render Pipeline)
    # Rasterizes Gaussians as quads and performs alpha blending
    # This is the render-bound stage on high-end GPUs
    frame = gpu_render_rasterize(
        sorted_gaussians,
        screen_width,
        screen_height
    )
    
    return frame

def gpu_compute_preprocess(gaussians, view_mat, proj_mat, w, h):
    """
    GPU Compute Shader for pre-processing.
    Transforms 3D Gaussians to 2D screen-space quads.
    """
    # For each Gaussian i:
    # 1. Transform position p_i to camera space: p_cam = view_mat * p_i
    # 2. Project to screen space: p_screen = proj_mat * p_cam
    # 3. Compute 2D covariance T_i from 3D covariance C_i and projection Jacobian J
    #    T_i = J * C_i * J^T
    # 4. Determine tile ID based on p_screen and tile size
    # 5. Compute bounding box for the Gaussian in screen space
    # 6. Filter out Gaussians outside the view frustum
    pass

def gpu_compute_radix_sort_wait_free(gaussians):
    """
    GPU Compute Shader for wait-free radix sort.
    Sorts Gaussians by tile ID, then by depth.
    """
    # Implementation of a lock-free radix sort algorithm
    # Uses atomic operations or workgroup barriers appropriately for WebGPU
    # Avoids spin-wait loops that cause stalls on Apple M-series chips
    pass

def gpu_render_rasterize(sorted_gaussians, w, h):
    """
    GPU Render Pipeline for rasterization.
    Renders Gaussians as screen-space quads with alpha blending.
    """
    # Vertex Shader:
    # For each Gaussian, generate a quad centered at p_screen with size derived from T_i
    # Fragment Shader:
    # For each pixel, evaluate the 2D Gaussian function
    # Color = SH_evaluation(sh_coeffs, view_dir)
    # Alpha = opacity * exp(-0.5 * (x^T * T_i^-1 * x))
    # Blend using front-to-back alpha blending
    pass
```

### Parameters and Typical Values
*   **Tile Size**: Typically 16x16 or 32x32 pixels, balancing sorting overhead and rasterization efficiency [S1, unofficial].
*   **Radix Sort Bits**: Standard 8-bit or 16-bit radix passes depending on the range of tile IDs [S1, unofficial].
*   **Scene Complexity**: Tested scenes range from 0.3M splats (van gogh) to 5.8M splats (garden) [S1, unofficial].

### Edge Cases and Pitfalls
*   **Apple M-series Spin-waiting**: Implementations using spin-wait-based sorting (e.g., KeKsBoTer/web-splat) suffer from significant busy-waiting on Apple M1/M2/M4 chips due to lack of workgroup execution order guarantees [S1, unofficial]. WebSplatter's wait-free design avoids this [S1, unofficial].
*   **Memory Exhaustion on Mobile**: Baselines like mkkellogg/GaussianSplatsD crash on iPhone 15 Pro Max due to excessive memory requirements [S1, unofficial]. WebSplatter's optimized memory usage prevents this [S1, unofficial].
*   **Firefox WebGPU Instability**: Firefox may block WebGPU or report unreliable timestamp queries on certain hardware (e.g., Intel NUC, Lenovo MX350) [S1, unofficial]. WebSplatter's compatibility is tested primarily on Chrome/Chromium-based browsers [S1, unofficial].
*   **Pre-processing Bottleneck on Low-end GPUs**: On devices like Nvidia MX350, the pre-processing stage (scanning raw splats) becomes the bottleneck due to limited memory throughput, consuming up to 47.2% of frame time [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: The sorting stage is $O(N \log N)$ or $O(N)$ depending on the radix sort implementation, but WebSplatter's wait-free sort ensures it never becomes the bottleneck [S1, unofficial]. Pre-processing is $O(N)$ [S1, unofficial]. Rendering is $O(N)$ in terms of splat count, but rasterization overhead depends on coverage [S1, unofficial].
*   **Memory Complexity**: $O(N)$ for storing Gaussian attributes and intermediate buffers [S1, unofficial]. WebSplatter's memory footprint is optimized to run on devices with limited VRAM, such as mobile phones [S1, unofficial].
*   **Measured Performance**:
    *   **Nvidia RTX 3070**: Median frame time of 9.497 ms on the garden scene (5.8M splats), a 1.52× speedup over the fastest competing WebGPU implementation [S1, unofficial].
    *   **MacBook Air M4**: 1.06× faster than baseline; P99 latency of 116.851 ms with a standard deviation of 22.303 ms on the garden scene [S1, unofficial].
    *   **Redmi K70 Pro**: 1.18× faster than baseline; renders truck scene (2.5M splats) in 52.7 ms without crashing [S1, unofficial].
    *   **Intel NUC iGPU**: 2.26× faster than baseline [S1, unofficial].
    *   **Scalability**: On Nvidia MX350, scaling from 0.3M to 5.8M splats increases frame time by ~143 ms, compared to >280 ms for KeKsBoTer/web-splat (2.1× speedup on largest scene) [S1, unofficial].
*   **Bottleneck Analysis**:
    *   **High-end GPUs** (RTX 3070, Apple M-series, Redmi K70 Pro): Render-bound. Rasterization consumes 51.8% to 62.6% of frame time [S1, unofficial].
    *   **Low-end GPUs** (Nvidia MX350, Intel NUC iGPU): Pre-process-bound. Pre-processing consumes up to 47.2% of frame time due to limited memory throughput [S1, unofficial].
    *   **Sort-bound**: WebSplatter never becomes sort-bound across all tested hardware [S1, unofficial].
*   **Strengths**:
    *   Cross-device compatibility (works on high-end desktops, laptops, and mobile phones) [S1, unofficial].
    *   Stability (no crashes on mobile devices where baselines fail) [S1, unofficial].
    *   Visual fidelity comparable to native CUDA implementation [S1, unofficial].
    *   Low variance in frame times, mitigating micro-stuttering [S1, unofficial].
*   **Limitations**:
    *   Performance on low-end GPUs is limited by memory throughput in the pre-processing stage [S1, unofficial].
    *   Relies on WebGPU support, which may be blocked or unstable in some browsers (e.g., Firefox) on certain hardware [S1, unofficial].

## Background
WebSplatter was introduced in the paper "WebSplatter: Enabling Cross-Device Efficient Gaussian Splatting in Web Browsers via WebGPU" [S1, unofficial]. It addresses the limitations of earlier web-based 3DGS renderers, which suffered from poor performance on Apple Silicon, instability on mobile devices, and high memory consumption [S1, unofficial]. The method builds on the 3DGS algorithm [S1, unofficial] and leverages the WebGPU API for hardware-accelerated rendering [S1, unofficial].

## Sources
[S1] WebSplatter: Enabling Cross-Device Efficient Gaussian Splatting in Web Browsers via WebGPU, https://doi.org/10.48550/arxiv.2602.03207, fetched 2026-10-26 (unofficial)
