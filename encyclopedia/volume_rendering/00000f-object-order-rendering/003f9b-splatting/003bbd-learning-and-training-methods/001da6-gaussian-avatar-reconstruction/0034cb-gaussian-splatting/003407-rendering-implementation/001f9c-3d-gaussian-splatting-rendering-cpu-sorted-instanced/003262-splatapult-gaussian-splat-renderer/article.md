# Splatapult Gaussian Splat Renderer

Also called: 3D Gaussian Splatting Rendering (SplatNode pipeline), Gaussian Splatting radiance-field cloud render, SplatNode pipeline, SplatNode render

## Overview
Splatapult is a display-only renderer for trained 3D Gaussian Splatting scenes that projects 3D Gaussian distributions into 2D screen-space quads, sorts them by camera-space depth on the GPU (or CPU fallback), and renders them using hardware-instanced drawing with standard alpha blending [S1, unofficial][S2, unofficial]. It distinguishes itself from sibling renderers by providing a complete, standalone implementation of the GPU-based multi-radix sorting pipeline required for correct depth ordering in instanced rendering, supporting VR via OpenXR and optional full spherical harmonic (SH) color evaluation [S1, unofficial][S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- A 3D Gaussian Splatting scene file (PLY format) containing Gaussian positions, covariances, opacities, and spherical harmonic coefficients [S1, unofficial][S2, unofficial].
- Camera matrices: Model-View-Projection matrix ($MVP$), Model-View matrix ($MV$), and projection matrix ($P$) [S1, unofficial].
- Viewport dimensions and near/far plane distances [S1, unofficial].

**Outputs:**
- A rasterized 2D image with correct depth ordering and alpha blending [S1, unofficial][S2, unofficial].
- Optional VR stereo frames [S1, unofficial].

### Data Structures
The renderer manages several GPU buffers for the sorting pipeline. All buffers use `std430` layout unless specified [S1, unofficial].

1.  **Position Buffer (`posBuffer`)**:
    -   Type: `GL_SHADER_STORAGE_BUFFER`
    -   Layout: Array of `vec4` (x, y, z, 1.0). Size: $N$ (number of Gaussians).
    -   Purpose: Stores 3D positions of all Gaussians [S1, unofficial].

2.  **Key Buffers (`keyBuffer`, `keyBuffer2`)**:
    -   Type: `GL_SHADER_STORAGE_BUFFER`
    -   Layout: Array of `uint32_t`. Size: $N$.
    -   Purpose: Stores the sorted depth keys (projected depth). `keyBuffer2` is used for ping-ponging during the sort [S1, unofficial].

3.  **Value Buffers (`valBuffer`, `valBuffer2`)**:
    -   Type: `GL_SHADER_STORAGE_BUFFER`
    -   Layout: Array of `uint32_t` (indices). Size: $N$.
    -   Purpose: Stores the original indices of the Gaussians, permuted alongside the keys. `valBuffer2` is used for ping-ponging [S1, unofficial].

4.  **Histogram Buffer (`histogramBuffer`)**:
    -   Type: `GL_SHADER_STORAGE_BUFFER`
    -   Layout: Array of `uint32_t`. Size: $256 \times W$, where $W$ is the number of workgroups.
    -   Purpose: Stores per-workgroup bin counts for the radix sort [S1, unofficial].

5.  **Atomic Counter Buffer (`atomicCounterBuffer`)**:
    -   Type: `GL_ATOMIC_COUNTER_BUFFER`
    -   Layout: Single `uint`.
    -   Purpose: Stores the count of visible Gaussians after the pre-sort culling stage [S1, unofficial].

6.  **VAO Element Buffer**:
    -   Type: `GL_ELEMENT_ARRAY_BUFFER`
    -   Layout: Array of `uint32_t` (indices). Size: $N$.
    -   Purpose: Holds the final sorted indices for instanced drawing [S1, unofficial].

### Numbered Steps

1.  **Initialization**:
    -   Load vertex (`splat_vert.glsl`), geometry (`splat_geom.glsl`), and fragment (`splat_frag.glsl`) shaders [S1, unofficial]. Apply defines: `FRAMEBUFFER_SRGB` if sRGB is enabled, `FULL_SH` if full spherical harmonics are present [S1, unofficial].
    -   Load the pre-sort compute shader (`presort_compute.glsl`) [S1, unofficial].
    -   Check for `GLEW_KHR_shader_subgroup` support [S1, unofficial]. If present and not overridden by `useRgcSortOverride`, load multi-radix sort shaders (`multi_radixsort.glsl`, `multi_radixsort_histograms.glsl`) [S1, unofficial]. Otherwise, initialize the CPU-based `rgc::radix_sort::sorter` [S1, unofficial].
    -   Populate `posBuffer` with Gaussian positions $(x, y, z, 1.0)$ [S1, unofficial].
    -   Allocate `keyBuffer`, `valBuffer`, and `posBuffer` [S1, unofficial]. If using GPU sort, also allocate `keyBuffer2`, `valBuffer2`, `histogramBuffer`, and `atomicCounterBuffer` [S1, unofficial].

2.  **Pre-sort (Culling and Key Generation)**:
    -   Bind `preSortProg` [S1, unofficial].
    -   Set uniforms: `modelViewProj` ($MVP$), `nearFar` (near and far plane distances), `keyMax` ($2^{32}-1$) [S1, unofficial].
    -   Reset `atomicCounterBuffer` to 0 [S1, unofficial].
    -   Bind buffers: `posBuffer` (slot 0, read-only), `keyBuffer` (slot 1, write-only), `valBuffer` (slot 2, write-only), `atomicCounterBuffer` (slot 4) [S1, unofficial].
    -   Dispatch compute shader with local size 256: `glDispatchCompute(ceil(N / 256), 1, 1)` [S1, unofficial].
    -   Execute barrier: `GL_SHADER_STORAGE_BARRIER_BIT | GL_ATOMIC_COUNTER_BARRIER_BIT` [S1, unofficial].
    -   *Note*: The pre-sort shader projects each Gaussian, checks visibility against the view frustum and near/far planes, and writes the projected depth as a 32-bit integer key to `keyBuffer` and the original index to `valBuffer` [S1, unofficial]. Visible Gaussians increment the atomic counter [S1, unofficial].

3.  **Get Count**:
    -   Read `atomicCounterBuffer` into host memory to obtain `sortCount` [S1, unofficial].
    -   Assert `sortCount <= N` [S1, unofficial].

4.  **Sorting**:
    -   **Option A: Multi-Radix Sort (GPU)**:
        -   Set `NUM_ELEMENTS = sortCount`, `NUM_WORKGROUPS = ceil(NUM_ELEMENTS / 1024)` [S1, unofficial].
        -   Loop for `i` from 0 to 3 (processing 4 bytes of the 32-bit key):
            1.  **Histogram Stage**:
                -   Bind `histogramProg`. Set `g_shift = 8 * i` [S1, unofficial].
                -   Bind `keyBuffer` (or `keyBuffer2` if `i` is odd) to slot 0, `histogramBuffer` to slot 1 [S1, unofficial].
                -   Dispatch compute: `glDispatchCompute(NUM_WORKGROUPS, 1, 1)` [S1, unofficial].
                -   Barrier: `GL_SHADER_STORAGE_BARRIER_BIT` [S1, unofficial].
            2.  **Sort Stage**:
                -   Bind `sortProg`. Set `g_shift = 8 * i` [S1, unofficial].
                -   Determine buffer bindings based on parity of `i`:
                    -   If `i` is even: Slot 0=`keyBuffer`, 1=`keyBuffer2`, 2=`valBuffer`, 3=`valBuffer2`, 4=`histogramBuffer` [S1, unofficial].
                    -   If `i` is odd: Slot 0=`keyBuffer2`, 1=`keyBuffer`, 2=`valBuffer2`, 3=`valBuffer`, 4=`histogramBuffer` [S1, unofficial].
                -   Dispatch compute: `glDispatchCompute(NUM_WORKGROUPS, 1, 1)` [S1, unofficial].
                -   Barrier: `GL_SHADER_STORAGE_BARRIER_BIT` [S1, unofficial].
    -   **Option B: CPU Radix Sort**:
        -   Call `sorter->sort(keyBuffer->GetObj(), valBuffer->GetObj(), sortCount)` [S1, unofficial].

5.  **Copy Sorted Indices**:
    -   If using multi-radix sort and `NUM_BYTES` (4) is even, bind `valBuffer` to `GL_COPY_READ_BUFFER` [S1, unofficial]. If odd, bind `valBuffer2` [S1, unofficial].
    -   Bind `splatVao->GetElementBuffer()` to `GL_COPY_WRITE_BUFFER` [S1, unofficial].
    -   Perform `glCopyBufferSubData` to transfer the sorted indices from the value buffer to the VAO's element buffer [S1, unofficial].

6.  **Rendering**:
    -   Bind the VAO with the sorted element buffer [S1, unofficial].
    -   Issue an instanced draw call using the geometry shader to project Gaussians into screen-space quads and the fragment shader to evaluate color (with optional SH) [S1, unofficial][S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def splatapult_render(num_points, use_gpu_sort, sort_count):
    # Initialization
    pos_buffer = [None] * num_points
    key_buffer = [0] * num_points
    val_buffer = [0] * num_points
    
    if use_gpu_sort:
        key_buffer2 = [0] * num_points
        val_buffer2 = [0] * num_points
        num_workgroups = (sort_count + 1023) // 1024
        histogram_buffer = [0] * (256 * num_workgroups)
        atomic_counter = [0]
        
        # Pre-sort (Culling)
        # glDispatchCompute(((num_points + 255) / 256), 1, 1)
        dispatch_compute(ceil(num_points / 256), 1, 1)
        barrier(ShaderStorage | AtomicCounter)
        
        sort_count = atomic_counter[0]
        assert sort_count <= num_points
        
        # Multi-Radix Sort
        for i in range(4):
            shift = 8 * i
            # Histogram Stage
            bind_buffer(hist_prog, 0, key_buffer if i % 2 == 0 else key_buffer2, READ_ONLY)
            bind_buffer(hist_prog, 1, histogram_buffer, WRITE_ONLY)
            dispatch_compute(num_workgroups, 1, 1)
            barrier(ShaderStorage)
            
            # Sort Stage
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
            dispatch_compute(num_workgroups, 1, 1)
            barrier(ShaderStorage)
    else:
        # CPU Radix Sort
        # sorter->sort(keyBuffer->GetObj(), valBuffer->GetObj(), sortCount)
        cpu_radix_sort(key_buffer, val_buffer, sort_count)

    # Copy Sorted Indices
    # If using multi-radix sort and NUM_BYTES is odd, bind valBuffer2, else bind valBuffer.
    # NUM_BYTES is 4 (even), so always use valBuffer.
    read_buffer = val_buffer
    bind_buffer(GL_COPY_READ_BUFFER, read_buffer)
    bind_buffer(GL_COPY_WRITE_BUFFER, vao_element_buffer)
    gl_copy_buffer_sub_data(GL_COPY_READ_BUFFER, GL_COPY_WRITE_BUFFER, 0, 0, sort_count * 4)

    # Render
    bind_vao(vao)
    draw_elements_instanced(GL_TRIANGLE_STRIP, 4, GL_UNSIGNED_INT, 0, sort_count)

# Helper definitions
def dispatch_compute(x, y, z): pass
def barrier(flags): pass
def bind_buffer(prog, slot, buf, access): pass
def cpu_radix_sort(keys, values, count): pass
def gl_copy_buffer_sub_data(src, dst, src_offset, dst_offset, size): pass
def bind_vao(vao): pass
def draw_elements_instanced(mode, primcount, type, indices, drawcount): pass
READ_ONLY, WRITE_ONLY, ShaderStorage, AtomicCounter, GL_COPY_READ_BUFFER, GL_COPY_WRITE_BUFFER = 0, 1, 4, 8, 32, 33
```

### Parameters
-   `NUM_BLOCKS_PER_WORKGROUP`: Constant set to 1024. Used to calculate the number of workgroups for GPU sort dispatch: `NUM_WORKGROUPS = (NUM_ELEMENTS + 1023) / 1024` [S1, unofficial].
-   `isFramebufferSRGBEnabled`: Boolean flag. If true, enables `#define FRAMEBUFFER_SRGB` in shaders to trigger sRGB framebuffer conversion, reducing color banding [S1, unofficial].
-   `gaussianCloud->HasFullSH()`: Boolean flag. If true, enables `#define FULL_SH` in shaders to evaluate full spherical harmonics [S1, unofficial].
-   `useRgcSortOverride`: Boolean flag. If true, forces the use of the CPU-based `rgc::radix_sort` even if GPU subgroup support is available [S1, unofficial].
-   `NUM_BYTES`: Set to 4 for 32-bit sorting [S1, unofficial].
-   `splatCount`: Controls the number of splats drawn (0..count), used for LOD or reveal hooks. Defaults to the full cloud count [S2, unofficial].
-   `cameraPositionProvider`: A callback returning the live camera world position, fed to the node's painter's sort each frame [S2, unofficial].

### Edge Cases and Pitfalls
-   **24-bit Radix Sort Artifacts**: The source notes that 24-bit radix sort produces artifacts on some datasets, so 32-bit sort (`NUM_BYTES=4`) is used by default [S1, unofficial].
-   **Hardware Support**: GPU multi-radix sort requires `GLEW_KHR_shader_subgroup` support. If not present, it falls back to CPU sort [S1, unofficial].
-   **Buffer Swapping**: The multi-radix sort implementation uses ping-ponging between `keyBuffer`/`keyBuffer2` and `valBuffer`/`valBuffer2` [S1, unofficial]. The final sorted indices are in `valBuffer` if the number of bytes (4) is even, or `valBuffer2` if odd [S1, unofficial]. The copy step must account for this [S1, unofficial].
-   **Within-batch alpha-blend order must be stable to prevent splat popping during slow camera orbits; the implementation guarantees this stability [S2, unofficial].
-   **SplatCloud is null while loading; the node declaration must handle the null case until the cloud is ready [S2, unofficial].

## Operating characteristics
-   **Time Complexity**: Sorting dominates [G].
-   **Memory Complexity**: $O(N)$ for positions, keys, values, and histograms [G].
-   **Measured Performance**: Not given in the sources.
-   **Scaling**: Scales well with $N$ on GPU. CPU sort scales with CPU cores but is bottlenecked by memory bandwidth [G].
-   **Strengths**: Correct depth ordering for alpha blending; VR support; optional full SH rendering; flexible sorting backend [S1, unofficial][S2, unofficial].
-   **Limitations**: Requires GPU for fast sorting (or falls back to slower CPU sort); 32-bit sort may be slightly slower than 24-bit but avoids artifacts; instanced rendering may have overhead for very small numbers of splats [S1, unofficial].
-   **When to Use**: When correct depth ordering is critical for alpha blending in 3DGS scenes, and VR or full SH rendering is desired [S1, unofficial][S2, unofficial].
-   **When Not to Use**: When performance is paramount and approximate sorting (e.g., screen-space order) is acceptable, or when hardware lacks GPU subgroup support and CPU sort is too slow [S1, unofficial].

## Background
Splatapult was created by Anthony J. Thibault and released as an open-source program to display 3D Gaussian Splatting files (PLY) [S1, unofficial]. It is based on the implementation of Intel's Embree radix sort, adapted for Vulkan/OpenGL compute shaders [S1, unofficial]. The project includes build scripts for CMake and vcpkg, and supports various rendering options like sRGB and full SH [S1, unofficial]. It is used in Android, Apple, Web, Flutter, and React Native SDKs for real-time on-device Gaussian Splatting rendering [S2, unofficial].

## Sources
[S1] hyperlogic/splatapult, https://github.com/hyperlogic/splatapult, fetched 2026-10-27 (unofficial)
[S2] feat(samples): splat-preview demo — first on-device SplatNode render ( #…, https://github.com/sceneview/sceneview/commit/5682de62179014b063bdff5529bedb8638385f03, fetched 2026-10-29 (unofficial)
