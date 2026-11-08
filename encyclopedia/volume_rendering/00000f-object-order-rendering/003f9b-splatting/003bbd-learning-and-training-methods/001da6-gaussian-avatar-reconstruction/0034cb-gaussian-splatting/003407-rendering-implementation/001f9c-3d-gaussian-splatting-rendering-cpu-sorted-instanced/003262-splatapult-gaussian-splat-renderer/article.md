# Splatapult Gaussian Splat Renderer

Also called: Splatapult

## Overview
Splatapult is a CPU-sorted, instanced renderer for 3D Gaussian Splatting (3DGS) scenes that performs depth sorting on the GPU using a multi-radix sort algorithm or falls back to a CPU-based radix sort. It projects 3D Gaussians into screen-space quads and renders them using standard alpha blending, supporting VR via OpenXR and optional full spherical harmonic (SH) color evaluation. It distinguishes itself from sibling renderers by providing a complete, standalone implementation of the GPU-based sorting pipeline required for correct depth ordering in instanced rendering, rather than relying on external libraries or CPU-only sorting.

## How it works
### Inputs and Outputs
**Inputs:**
- A 3D Gaussian Splatting scene file (PLY format) containing Gaussian positions, covariances, opacities, and spherical harmonic coefficients.
- Camera matrices: Model-View-Projection matrix ($MVP$), Model-View matrix ($MV$), and projection matrix ($P$).
- Viewport dimensions and near/far plane distances.

**Outputs:**
- A rasterized 2D image with correct depth ordering and alpha blending.
- Optional VR stereo frames.

### Data Structures
The renderer manages several GPU buffers for the sorting pipeline. All buffers use `std430` layout unless specified.

1.  **Position Buffer (`posBuffer`)**:
    -   Type: `GL_SHADER_STORAGE_BUFFER`
    -   Layout: Array of `vec4` (x, y, z, 1.0). Size: $N$ (number of Gaussians).
    -   Purpose: Stores 3D positions of all Gaussians.

2.  **Key Buffers (`keyBuffer`, `keyBuffer2`)**:
    -   Type: `GL_SHADER_STORAGE_BUFFER`
    -   Layout: Array of `uint32_t`. Size: $N$.
    -   Purpose: Stores the sorted depth keys (projected depth). `keyBuffer2` is used for ping-ponging during the sort.

3.  **Value Buffers (`valBuffer`, `valBuffer2`)**:
    -   Type: `GL_SHADER_STORAGE_BUFFER`
    -   Layout: Array of `uint32_t` (indices). Size: $N$.
    -   Purpose: Stores the original indices of the Gaussians, permuted alongside the keys. `valBuffer2` is used for ping-ponging.

4.  **Histogram Buffer (`histogramBuffer`)**:
    -   Type: `GL_SHADER_STORAGE_BUFFER`
    -   Layout: Array of `uint32_t`. Size: $256 \times W$, where $W$ is the number of workgroups.
    -   Purpose: Stores per-workgroup bin counts for the radix sort.

5.  **Atomic Counter Buffer (`atomicCounterBuffer`)**:
    -   Type: `GL_ATOMIC_COUNTER_BUFFER`
    -   Layout: Single `uint`.
    -   Purpose: Stores the count of visible Gaussians after the pre-sort culling stage.

6.  **VAO Element Buffer**:
    -   Type: `GL_ELEMENT_ARRAY_BUFFER`
    -   Layout: Array of `uint32_t` (indices). Size: $N$.
    -   Purpose: Holds the final sorted indices for instanced drawing.

### Numbered Steps

1.  **Initialization**:
    -   Load vertex (`splat_vert.glsl`), geometry (`splat_geom.glsl`), and fragment (`splat_frag.glsl`) shaders. Apply defines: `FRAMEBUFFER_SRGB` if sRGB is enabled, `FULL_SH` if full spherical harmonics are present.
    -   Load the pre-sort compute shader (`presort_compute.glsl`).
    -   Check for `GLEW_KHR_shader_subgroup` support. If present and not overridden by `useRgcSortOverride`, load multi-radix sort shaders (`multi_radixsort.glsl`, `multi_radixsort_histograms.glsl`). Otherwise, initialize the CPU-based `rgc::radix_sort::sorter`.
    -   Populate `posBuffer` with Gaussian positions $(x, y, z, 1.0)$.
    -   Allocate `keyBuffer`, `valBuffer`, and `posBuffer`. If using GPU sort, also allocate `keyBuffer2`, `valBuffer2`, `histogramBuffer`, and `atomicCounterBuffer`.

2.  **Pre-sort (Culling and Key Generation)**:
    -   Bind `preSortProg`.
    -   Set uniforms: `modelViewProj` ($MVP$), `nearFar` (near and far plane distances), `keyMax` ($2^{32}-1$).
    -   Reset `atomicCounterBuffer` to 0.
    -   Bind buffers: `posBuffer` (slot 0, read-only), `keyBuffer` (slot 1, write-only), `valBuffer` (slot 2, write-only), `atomicCounterBuffer` (slot 4).
    -   Dispatch compute shader with local size 256: `glDispatchCompute(ceil(N / 256), 1, 1)`.
    -   Execute barrier: `GL_SHADER_STORAGE_BARRIER_BIT | GL_ATOMIC_COUNTER_BARRIER_BIT`.
    -   *Note*: The pre-sort shader projects each Gaussian, checks visibility against the view frustum and near/far planes, and writes the projected depth as a 32-bit integer key to `keyBuffer` and the original index to `valBuffer`. Visible Gaussians increment the atomic counter.

3.  **Get Count**:
    -   Read `atomicCounterBuffer` into host memory to obtain `sortCount`.
    -   Assert `sortCount <= N`.

4.  **Sorting**:
    -   **Option A: Multi-Radix Sort (GPU)**:
        -   Set `NUM_ELEMENTS = sortCount`, `NUM_WORKGROUPS = ceil(NUM_ELEMENTS / 1024)`.
        -   Loop for `i` from 0 to 3 (processing 4 bytes of the 32-bit key):
            1.  **Histogram Stage**:
                -   Bind `histogramProg`. Set `g_shift = 8 * i`.
                -   Bind `keyBuffer` (or `keyBuffer2` if `i` is odd) to slot 0, `histogramBuffer` to slot 1.
                -   Dispatch compute: `glDispatchCompute(NUM_WORKGROUPS, 1, 1)`.
                -   Barrier: `GL_SHADER_STORAGE_BARRIER_BIT`.
            2.  **Sort Stage**:
                -   Bind `sortProg`. Set `g_shift = 8 * i`.
                -   Determine buffer bindings based on parity of `i`:
                    -   If `i` is even: Slot 0=`keyBuffer`, 1=`keyBuffer2`, 2=`valBuffer`, 3=`valBuffer2`, 4=`histogramBuffer`.
                    -   If `i` is odd: Slot 0=`keyBuffer2`, 1=`keyBuffer`, 2=`valBuffer2`, 3=`valBuffer`, 4=`histogramBuffer`.
                -   Dispatch compute: `glDispatchCompute(NUM_WORKGROUPS, 1, 1)`.
                -   Barrier: `GL_SHADER_STORAGE_BARRIER_BIT`.
    -   **Option B: CPU Radix Sort**:
        -   Call `sorter->sort(keyBuffer->GetObj(), valBuffer->GetObj(), sortCount)`.

5.  **Copy Sorted Indices**:
    -   If using multi-radix sort and `NUM_BYTES` (4) is even, bind `valBuffer` to `GL_COPY_READ_BUFFER`. If odd, bind `valBuffer2`.
    -   Bind `splatVao->GetElementBuffer()` to `GL_COPY_WRITE_BUFFER`.
    -   Perform `glCopyBufferSubData` to transfer the sorted indices from the value buffer to the VAO's element buffer.
    -   *Note*: The source code snippet ends here, implying this copy operation completes the preparation for rendering.

6.  **Rendering**:
    -   Bind the VAO with the sorted element buffer.
    -   Issue an instanced draw call using the geometry shader to project Gaussians into screen-space quads and the fragment shader to evaluate color (with optional SH).

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def splatapult_render(scene, camera, viewport):
    # 1. Initialization
    num_points = scene.num_gaussians
    pos_buffer = allocate_buffer(num_points, vec4)
    key_buffer = allocate_buffer(num_points, uint32)
    val_buffer = allocate_buffer(num_points, uint32)
    
    # Check GPU sort capability
    use_gpu_sort = check_shader_subgroup_support() and not useRgcSortOverride
    
    if use_gpu_sort:
        key_buffer2 = allocate_buffer(num_points, uint32)
        val_buffer2 = allocate_buffer(num_points, uint32)
        histogram_buffer = allocate_buffer(num_points, uint32) # Size depends on workgroups
        atomic_counter = allocate_atomic_counter()
        
        # Load GPU shaders
        pre_sort_prog = load_compute_shader("presort_compute.glsl")
        hist_prog = load_compute_shader("multi_radixsort_histograms.glsl")
        sort_prog = load_compute_shader("multi_radixsort.glsl")
    else:
        # Load CPU sorter
        cpu_sorter = rgc_radix_sort.Sorter()

    # Fill position buffer
    for i in range(num_points):
        pos_buffer[i] = vec4(scene.gaussians[i].position, 1.0)

    # 2. Pre-sort (Culling)
    bind(pre_sort_prog)
    set_uniform(pre_sort_prog, "modelViewProj", camera.mvp)
    set_uniform(pre_sort_prog, "nearFar", camera.near_far)
    set_uniform(pre_sort_prog, "keyMax", 0xFFFFFFFF)
    
    bind_buffer(pre_sort_prog, 0, pos_buffer, READ_ONLY)
    bind_buffer(pre_sort_prog, 1, key_buffer, WRITE_ONLY)
    bind_buffer(pre_sort_prog, 2, val_buffer, WRITE_ONLY)
    bind_buffer(pre_sort_prog, 4, atomic_counter, ATOMIC_COUNTER)
    
    dispatch_compute(pre_sort_prog, ceil(num_points / 256), 1, 1)
    barrier(ShaderStorage | AtomicCounter)

    # 3. Get Count
    sort_count = read_atomic_counter(atomic_counter)
    assert sort_count <= num_points

    # 4. Sorting
    if use_gpu_sort:
        num_workgroups = ceil(sort_count / 1024)
        for i in range(4): # 4 bytes for uint32
            shift = 8 * i
            
            # Histogram Stage
            bind(hist_prog)
            set_uniform(hist_prog, "g_shift", shift)
            set_uniform(hist_prog, "g_num_workgroups", num_workgroups)
            
            # Bind input key buffer based on parity
            if i % 2 == 0:
                bind_buffer(hist_prog, 0, key_buffer, READ_ONLY)
            else:
                bind_buffer(hist_prog, 0, key_buffer2, READ_ONLY)
            bind_buffer(hist_prog, 1, histogram_buffer, WRITE_ONLY)
            
            dispatch_compute(hist_prog, num_workgroups, 1, 1)
            barrier(ShaderStorage)
            
            # Sort Stage
            bind(sort_prog)
            set_uniform(sort_prog, "g_shift", shift)
            set_uniform(sort_prog, "g_num_workgroups", num_workgroups)
            set_uniform(sort_prog, "g_num_elements", sort_count)
            
            # Ping-pong buffers
            if i % 2 == 0:
                bind_buffer(sort_prog, 0, key_buffer, READ_ONLY)
                bind_buffer(sort_prog, 1, key_buffer2, WRITE_ONLY)
                bind_buffer(sort_prog, 2, val_buffer, READ_ONLY)
                bind_buffer(sort_prog, 3, val_buffer2, WRITE_ONLY)
            else:
                bind_buffer(sort_prog, 0, key_buffer2, READ_ONLY)
                bind_buffer(sort_prog, 1, key_buffer, WRITE_ONLY)
                bind_buffer(sort_prog, 2, val_buffer2, READ_ONLY)
                bind_buffer(sort_prog, 3, val_buffer, WRITE_ONLY)
            bind_buffer(sort_prog, 4, histogram_buffer, READ_ONLY)
            
            dispatch_compute(sort_prog, num_workgroups, 1, 1)
            barrier(ShaderStorage)

    else:
        # CPU Sort
        cpu_sorter.sort(key_buffer, val_buffer, sort_count)

    # 5. Copy Sorted Indices to VAO
    if use_gpu_sort:
        if 4 % 2 == 0: # Even number of bytes processed
            read_buffer = val_buffer
        else:
            read_buffer = val_buffer2
            
        bind_buffer(GL_COPY_READ_BUFFER, read_buffer)
        bind_buffer(GL_COPY_WRITE_BUFFER, vao_element_buffer)
        gl_copy_buffer_sub_data(GL_COPY_READ_BUFFER, GL_COPY_WRITE_BUFFER, 0, 0, sort_count * 4)
    else:
        # CPU sort already wrote to val_buffer, copy to VAO
        bind_buffer(GL_COPY_READ_BUFFER, val_buffer)
        bind_buffer(GL_COPY_WRITE_BUFFER, vao_element_buffer)
        gl_copy_buffer_sub_data(GL_COPY_READ_BUFFER, GL_COPY_WRITE_BUFFER, 0, 0, sort_count * 4)

    # 6. Render
    bind_vao(vao)
    set_uniform(render_prog, "modelViewProj", camera.mvp)
    # ... set other uniforms ...
    draw_elements_instanced(GL_TRIANGLE_STRIP, 4, GL_UNSIGNED_INT, 0, sort_count)
```

### Parameters
-   `NUM_BLOCKS_PER_WORKGROUP`: Constant set to 1024. Used to calculate the number of workgroups for GPU sort dispatch: `NUM_WORKGROUPS = (NUM_ELEMENTS + 1023) / 1024`.
-   `isFramebufferSRGBEnabled`: Boolean flag. If true, enables `#define FRAMEBUFFER_SRGB` in shaders to trigger sRGB framebuffer conversion, reducing color banding.
-   `gaussianCloud->HasFullSH()`: Boolean flag. If true, enables `#define FULL_SH` in shaders to evaluate full spherical harmonics.
-   `useRgcSortOverride`: Boolean flag. If true, forces the use of the CPU-based `rgc::radix_sort` even if GPU subgroup support is available.
-   `NUM_BYTES`: Set to 4 for 32-bit sorting. 24-bit sorting is avoided due to artifacts on some datasets [S1, unofficial].

### Edge Cases and Pitfalls
-   **24-bit Radix Sort Artifacts**: The source notes that 24-bit radix sort produces artifacts on some datasets, so 32-bit sort (`NUM_BYTES=4`) is used by default [S1, unofficial].
-   **Hardware Support**: GPU multi-radix sort requires `GLEW_KHR_shader_subgroup` support. If not present, it falls back to CPU sort [S1, unofficial].
-   **Buffer Swapping**: The multi-radix sort implementation uses ping-ponging between `keyBuffer`/`keyBuffer2` and `valBuffer`/`valBuffer2`. The final sorted indices are in `valBuffer` if the number of bytes (4) is even, or `valBuffer2` if odd. The copy step must account for this [S1, unofficial].

## Subtopics

### [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md)
Uses a tiled approach to rasterize splats, reducing overdraw and blending costs. Choose when rendering high-density scenes on modern GPUs with tile-based rendering support.

### [Gaussian Splat Rendering (Screen-Space Quad)](2075-gaussian-splat-rendering-screen-space-quad.md)
Renders splats as screen-space quads without geometry shaders, often using a different projection method. Choose for compatibility with renderers that do not support geometry shaders or for simpler pipeline integration.

### [Depth-First Splat Sorting](2850-depth-first-splat-sorting.md)
Sorts splats using a depth-first traversal of a spatial hierarchy (e.g., BVH) rather than a global sort. Choose when hierarchical culling is more efficient than global sorting for sparse scenes.

## Operating characteristics
-   **Time Complexity**: The sorting step dominates, with GPU multi-radix sort being $O(N \log N)$ but with a very low constant factor due to parallelism. CPU sort is also $O(N \log N)$ but slower due to CPU-GPU transfer and single-threaded (or less parallel) execution.
-   **Memory Complexity**: $O(N)$ for storing positions, keys, values, and histograms. The GPU sort requires additional buffers for ping-ponging and histograms.
-   **Measured Performance**: Not given in the sources.
-   **Scaling**: Scales well with the number of Gaussians on GPU due to parallel sorting. CPU sort scales linearly with CPU cores but is bottlenecked by memory bandwidth and single-threaded performance for large $N$.
-   **Strengths**: Correct depth ordering for alpha blending; VR support; optional full SH rendering; flexible sorting backend (GPU/CPU).
-   **Limitations**: Requires GPU for fast sorting (or falls back to slower CPU sort); 32-bit sort may be slightly slower than 24-bit but avoids artifacts; instanced rendering may have overhead for very small numbers of splats.
-   **When to Use**: When correct depth ordering is critical for alpha blending in 3DGS scenes, and VR or full SH rendering is desired.
-   **When Not to Use**: When performance is paramount and approximate sorting (e.g., screen-space order) is acceptable, or when hardware lacks GPU subgroup support and CPU sort is too slow.

## Background
Splatapult was created by Anthony J. Thibault and released as an open-source program to display 3D Gaussian Splatting files (PLY). It is based on the implementation of Intel's Embree radix sort, adapted for Vulkan/OpenGL compute shaders. The project includes build scripts for CMake and vcpkg, and supports various rendering options like sRGB and full SH. It is cited as an unofficial source [S1, unofficial].

## Sources
[S1] hyperlogic/splatapult, https://github.com/hyperlogic/splatapult, fetched 2026-10-27 (unofficial)
