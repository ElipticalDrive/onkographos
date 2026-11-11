# CUDA GPU Implementation

Also called: CUDA Raycast Backend, cvc::volren CUDA backend

## Overview
A CUDA-based direct volume ray casting backend that mirrors the CPU marcher semantically to achieve interactive performance on NVIDIA hardware [S1, unofficial]. It distinguishes itself from CPU implementations by leveraging parallel shader execution for ray traversal, sampling, and compositing, while maintaining image-level parity (within 1 LSB) with the reference implementation [S1, unofficial]. Unlike general-purpose GPU volume renderers, this specific implementation is currently limited to single-volume scenes with a fixed set of features (up to 8 isosurfaces/lights/cut planes) and relies on per-render host-to-device data uploads [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D volume data (scalar or SDF), transfer function (baked LUT), camera parameters, and rendering options (isosurface threshold, lighting, cut planes).
*   **Output:** 2D image buffer with accumulated color and depth.

**Data Structures:**
*   **Volume Data:** Stored in device memory, uploaded from host for each render pass [S1, unofficial].
*   **Transfer Function:** Baked Look-Up Table (LUT) stored in device memory, uploaded from host for each render pass [S1, unofficial].
*   **Spatial Search (MLS):** Dynamic redundant octree (Guennebaud et al.) stored in device memory for neighbor sampling in MLS-based rendering [S2, unofficial].
*   **Ray Stack:** Small stack stored in local memory to manage recursion for adaptive ray traversal [S2, unofficial].
*   **Marching Cubes Tables:** `__constant__` memory tables for isosurface extraction, tracked per device to avoid cross-GPU contamination [S1, unofficial].

**Numbered Steps:**
1.  **Backend Selection:** The renderer selects the backend based on configuration: `backend::cpu` (default), `backend::cuda` (demands GPU, fails if scene constraints not met), or `backend::automatic` (uses GPU if available and scene fits, falls back silently) [S1, unofficial].
2.  **Data Upload:** Volume data and LUT are uploaded from host to device (H2D) for each render frame [S1, unofficial].
3.  **Thread Launch:** Launch one CUDA thread per pixel in 16x16 blocks [S1, unofficial].
4.  **Ray Generation:** Each thread computes ray parameters from camera matrices and performs slab intersection to determine entry/exit points within the volume bounding box [S1, unofficial].
5.  **Ray Traversal:**
    *   **Isosurface Rendering:** Uses Amanatides-Woo DDA to traverse cells, enumerating every crossed cell to avoid silhouette holes [S1, unofficial].
    *   **General Rendering:** Supports both constant step size and adaptive algorithms. Adaptive traversal uses a small stack in local memory to manage recursion [S2, unofficial].
    *   **Neighbor Sampling:** For MLS-based rendering, uses the dynamic redundant octree to efficiently find neighbor samples [S2, unofficial].
6.  **Gradient Computation:** For SDF volumes, computes gradients using de Boor splines [S1, unofficial].
7.  **Volume Sampling:** Samples volume data using trilinear interpolation [S1, unofficial].
8.  **Classification:** Applies transfer functions using the baked LUT [S1, unofficial].
9.  **Lighting and Effects:** Applies gradient ramp, density window, and cut planes. Computes lighting using Blinn-Phong model [S1, unofficial].
10. **Compositing:** Composites samples front-to-back on a per-cell basis [S1, unofficial].
11. **Early Ray Termination:** Rays are terminated when accumulated opacity exceeds a threshold or a maximum hit count (e.g., 32 hits) is reached [S1, unofficial].
12. **Depth Latch:** Stores depth information for the final pixel [S1, unofficial].

**Formulas:**
*   **Compositing (Front-to-Back):**
    $$
    C'_{i} = C'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i C_i
    $$
    $$
    \alpha'_{i} = \alpha'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i
    $$
    Where $C_i$ and $\alpha_i$ are the color and opacity of the current sample, and $C'$ and $\alpha'$ are the accumulated color and opacity [S1, unofficial].

**Parameters:**
*   **Block Size:** 16x16 threads per block [S1, unofficial].
*   **Max Hits:** 32 hits per ray cap (documented as unreachable in early versions, but enforced) [S1, unofficial].
*   **Isosurface Limit:** Up to 8 isosurfaces [S1, unofficial].
*   **Light/Cut Plane Limit:** Up to 8 lights and cut planes [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **NaN Handling:** Corner min/max folds must use `std::min`/`std::max` (keeps NaN accumulator) instead of `fminf`/`fmaxf` (IEEE, discards NaN) to correctly handle cells with NaN corners in the isovalue-bracket test [S1, unofficial].
*   **Multi-GPU Constant Memory:** `__constant__` tables (e.g., marching cubes) are per-device. Uploading once per process causes silent vanishing of isosurfaces on secondary GPUs. Tables must be tracked per device [S1, unofficial].
*   **Error State:** `cudaGetLastError()` reports the last error since it was last read. An unrelated handled failure can cause spurious fallbacks. The error state must be cleared before launching [S1, unofficial].
*   **DDA vs. Opacity Cutoff:** The DDA collects every crossing before compositing starts, so the 32-hit cap is not solely due to opacity cutoff [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# raycast.cu kernel structure (simplified)

__global__ void raycast_kernel(
    float* volume_data,
    float* lut,
    float* camera_matrix,
    float* output_color,
    float* output_depth,
    int width,
    int height,
    float isovalue,
    bool use_adaptive,
    Octree* octree // For MLS
) {
    int x = blockIdx.x * blockDim.x + threadIdx.x;
    int y = blockIdx.y * blockDim.y + threadIdx.y;

    if (x >= width || y >= height) return;

    // 1. Ray Generation
    Ray ray = generate_ray(camera_matrix, x, y, width, height);

    // 2. Slab Intersection
    Slab slab = intersect_slab(ray, volume_bounds);
    if (!slab.valid) return;

    float accumulated_color[3] = {0.0f, 0.0f, 0.0f};
    float accumulated_alpha = 0.0f;
    float depth = 0.0f;
    float t = slab.entry;

    // 3. Ray Traversal Loop
    while (t < slab.exit && accumulated_alpha < 0.95f) {
        // Adaptive step or constant step
        float step = use_adaptive ? adaptive_step(ray, t, octree) : constant_step;
        t += step;

        // 4. Sampling
        float3 sample_pos = ray.origin + ray.direction * t;
        float scalar = trilinear_sample(volume_data, sample_pos);

        // 5. Gradient (for SDF)
        float3 gradient = compute_gradient_de_boor(volume_data, sample_pos);

        // 6. Classification
        float4 tf_result = lookup_lut(lut, scalar, gradient);
        float alpha = tf_result.a;
        float3 color = tf_result.rgb;

        // 7. Lighting (Blinn-Phong)
        color = apply_blinn_phong(color, gradient, light_positions);

        // 8. Compositing
        if (alpha > 0.0f) {
            float3 new_color = accumulated_color + (1.0f - accumulated_alpha) * alpha * color;
            float new_alpha = accumulated_alpha + (1.0f - accumulated_alpha) * alpha;
            accumulated_color = new_color;
            accumulated_alpha = new_alpha;
            depth = t;
        }

        // 9. Early Ray Termination
        if (accumulated_alpha >= 0.95f) break;
    }

    // 10. Output
    output_color[(y * width + x) * 3 + 0] = accumulated_color[0];
    output_color[(y * width + x) * 3 + 1] = accumulated_color[1];
    output_color[(y * width + x) * 3 + 2] = accumulated_color[2];
    output_depth[y * width + x] = depth;
}
```

## Subtopics

### [Slab-based Raycasting](11383-slab-based-raycasting.md)
Slab-based raycasting is a CUDA GPU optimization for direct volume rendering that caches sampled voxels in shared memory as "slabs" to improve performance when volume data is accessed multiple times, such as for gradient calculation in Phong lighting. It sets itself apart from basic raycasting by trading increased shared memory overhead and reduced early ray termination granularity for cache hits on repeated voxel access. Unlike hardware texture caches, which benefit from large viewports where adjacent rays hit the same voxels, the slab cache is most efficient when the data set resolution is high compared to the viewport size.

### [GPU-accelerated ray casting](12256-gpu-accelerated-ray-casting.md)
GPU-accelerated ray casting is a direct volume rendering technique that offloads the computationally intensive tasks of ray-bounding box intersection, resampling, and compositing to the graphics processing unit (GPU) to achieve real-time interactive performance. It distinguishes itself from CPU-based implementations by leveraging parallel shader execution, which significantly reduces running time despite the underlying computational complexity of interpolation remaining constant for a given viewpoint.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels and the number of samples per ray. Adaptive traversal reduces the number of samples in empty space.
*   **Memory Complexity:** Linear with respect to the volume size and octree size (for MLS).
*   **Measured Performance:** On a GTX 1650 at 512^2 resolution, isosurface rendering takes 65 ms on CUDA vs 160 ms on CPU. Shaded TF + ramp + cut plane takes 113 ms on CUDA vs 270 ms on CPU [S1, unofficial]. One bunny on CUDA at 640x360 resolution: 14 ms per frame, 16.5 Mray/s [S1, unofficial].
*   **Strengths:** Significant speedup over CPU implementation (2-3x in tested scenarios). Image-level parity with CPU implementation (100% of pixels within 1 LSB).
*   **Limitations:** Currently limited to single-volume scenes (v1). Requires per-render H2D uploads. `__constant__` memory issues on multi-GPU setups if not handled correctly.
*   **When to Use:** When real-time or interactive performance is required on NVIDIA hardware and the scene fits within the single-volume constraint.
*   **When Not to Use:** When multi-volume rendering is required (falls back to CPU). When running on non-NVIDIA hardware.

## Background
The CUDA GPU Implementation is part of the `cvc::volren` library, specifically the `raycast.cu` module. It was developed to provide a high-performance alternative to the CPU-based ray caster, leveraging the parallel processing capabilities of NVIDIA GPUs. The implementation mirrors the CPU marcher semantically to ensure correctness and parity. It was introduced in the context of the `cvcGL VolRenNode` and `volren_bunny` demo [S1, unofficial]. The underlying ray casting principles are based on the emission-absorption optical model and hardware-accelerated sampling techniques described in [Direct volume ray casting](19-direct-volume-ray-casting.md) and [Hardware-Accelerated and GPU Implementations](9525-hardware-accelerated-and-gpu-implementations.md). The MLS-based neighbor sampling uses the dynamic redundant octree of Guennebaud et al. [S2, unofficial].

## Sources
[S1] cvcGL VolRenNode, CUDA raycast backend, and the volren_bunny demo, https://github.com/transfix/libcvc/pull/323, fetched 2026-10-25 (unofficial)
[S2] Volume MLS Ray Casting, https://miriah.github.io/publications/volume_mls.pdf, fetched 2026-10-29 (unofficial)
