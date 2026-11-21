# CUDA-OpenGL Interoperability

Also called: CUDA-GL Interop, CUDA-to-OpenGL PBO write, OpenGL-CUDA Interoperability via Pixel Buffer Object (PBO), PBO-based rendering

## Overview
CUDA-OpenGL Interoperability is a method for sharing GPU memory between CUDA computational kernels and the OpenGL rendering pipeline without CPU-mediated data transfers [S2][S4, unofficial][S5]. It solves the bottleneck where the CPU issues draw calls or copies data between host and device memory, which limits frame rates in real-time volume rendering [S2, S5]. Unlike traditional raster engines that rely on draw calls and shader binding, this method allows CUDA kernels to write directly to OpenGL resources (such as Pixel Buffer Objects or 3D textures) or read from them via mapped CUDA arrays, enabling zero-copy data sharing between simulation and rendering [S3, S4, unofficial][S5].

## How it works
The method establishes a bridge between the CUDA and OpenGL contexts using the `cudaGraphicsRegister` API family [S7, unofficial]. It supports two primary resource types: Pixel Buffer Objects (PBOs) for image output and OpenGL Textures (2D/3D) for data input or output via surface objects [S4, unofficial].

### Inputs and Outputs
- **Inputs:** OpenGL resource IDs (texture IDs, buffer IDs), OpenGL context handle, CUDA device ID.
- **Outputs:** CUDA graphics resource handles (`cudaGraphicsResource_t`), CUDA array views (`cudaArray_t`), CUDA surface objects (`cudaSurfaceObject_t`), or device pointers (`void*`) [S3, S6, S7, S9, unofficial].

### Data Structures and Layouts
- **CUDA Graphics Resource Handle:** An opaque handle returned by registration functions, used to map/unmap the resource [S3, unofficial]. Type: `cudaGraphicsResource_t`.
- **CUDA Array View:** A linear memory structure representing the OpenGL texture data, accessible via texture/surface references [S1]. Type: `cudaArray_t`.
- **CUDA Surface Object:** A writable view of a CUDA array, allowing direct per-voxel/pixel writes from kernels [S6, unofficial]. Type: `cudaSurfaceObject_t`.
- **CudaGlTexture3D Class Layout (Example Implementation):** `m_resource` (`cudaGraphicsResource_t*`), `m_width` (`int`), `m_height` (`int`), `m_depth` (`int`) [S3, unofficial].

### Numbered Steps
1. **Registration:** Create the OpenGL resource (PBO, 3D/2D Texture, or VBO) [S2, S5][S9, unofficial]. Register it with CUDA using `cudaGraphicsGLRegisterImage` for textures or `cudaGraphicsGLRegisterBuffer` for buffers [S3, S9, unofficial]. Specify flags: `cudaGraphicsRegisterFlagsSurfaceLoadStore` for writable textures, `cudaGraphicsMapFlagsReadOnly` for read-only textures, or `cudaGraphicsMapFlagsNone` for buffers [S3, S6, S9, unofficial].
2. **Mapping:** Call `cudaGraphicsMapResources` to gain CUDA access and synchronize contexts [S3, S6, S7, S9, unofficial]. For textures, call `cudaGraphicsSubResourceGetMappedArray` to retrieve the `cudaArray_t` [S3, S6, S7, unofficial]. Bind it to a CUDA texture object via `cudaBindTextureToArray` or to a surface object via `cudaBindSurfaceToArray` [S6, unofficial]. For buffers, call `cudaGraphicsResourceGetMappedPointer` to retrieve the raw device pointer [S9, unofficial].
3. **Computation:** Launch the CUDA kernel to read from mapped textures or write to mapped arrays/pointers [S2, S5][S7, S9, unofficial]. Synchronize with `cudaDeviceSynchronize()` to ensure completion before OpenGL access [S3, S7, unofficial].
4. **Unmapping & Rendering:** Call `cudaGraphicsUnmapResources` to release CUDA access [S3, S6, S7, S9, unofficial]. Display via PBO (bind to `GL_PIXEL_UNPACK_BUFFER` and draw a quad, or use an explicit FBO) [S2, S5], or bind the texture and draw a quad [S6, S10, unofficial]. For VBOs, bind buffers and call `glDrawArrays` [S9, unofficial].

### Pseudocode: PBO-Based Rendering Loop
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def init_cuda_gl_interop(pbo_id):
    # Register the OpenGL PBO with the CUDA runtime to enable zero-copy sharing
    cuda_resource = register_opengl_buffer_with_cuda(pbo_id, flags=MAP_FLAGS_NONE)
    return cuda_resource

def render_frame(cuda_resource, width, height, compute_kernel):
    # Gain CUDA access to the mapped OpenGL resource
    map_resources(1, cuda_resource, stream=0)
    
    # Retrieve the device pointer to the PBO memory
    device_ptr, bytes_count = get_mapped_pointer(cuda_resource)
    
    # Launch the volume rendering kernel
    block_dim = (16, 16, 1)
    grid_dim = ((width + 15) // 16, (height + 15) // 16, 1)
    compute_kernel(grid_dim, block_dim, device_ptr, width, height)
    
    # Ensure kernel completion before OpenGL access
    synchronize_device()
    
    # Release CUDA access to the resource
    unmap_resources(1, cuda_resource, stream=0)
    
    # Transfer PBO data to a display texture and render
    bind_texture_unit(GL_TEXTURE0)
    bind_texture(GL_TEXTURE_2D, display_texture_id)
    copy_pbo_to_texture(GL_TEXTURE_2D, 0, 0, 0, width, height, GL_RGBA, GL_UNSIGNED_BYTE, device_ptr)
    draw_screen_quad()
    swap_framebuffers()
```

### Parameters and Typical Values
- **Block Dimension:** Typically `dim3(16, 16, 1)` for 2D image rendering [S6, unofficial].
- **Grid Dimension:** Calculated as `(Width + block_dim.x) / block_dim.x` for x, `(Height + block_dim.y) / block_dim.y` for y, and `1` for z [S6, unofficial].
- **Interpolation Filter:** Hamming windowed-sinc filter with width 5 is used to minimize artifacts in resampling operations [S4, unofficial].
- **Zero-Padding:** 100% zero-padding increases memory requirements by a factor of 8 but suppresses overlapping artifacts in Fourier-based methods [S4, unofficial].

### Edge Cases and Pitfalls
- **Context Switching:** Direct access to CUDA address space memory from OpenGL is not possible; data must be transferred via shared buffers like PBOs [S4, unofficial]. Switching between off-screen OpenGL contexts and CUDA contexts must be handled carefully to avoid resource conflicts [S1][S4, unofficial].
- **Premature Access:** Accessing mapped VBO/PBO memory with OpenGL before unmapping causes program failure [S9, unofficial].
- **Error Handling:** If `cudaGraphicsSubResourceGetMappedArray` fails, the surface object is not created, preventing kernel launch. Code must check `cudaSuccess` before creating surfaces [S3, unofficial].
- **Performance:** Reading device memory via global memory is less performant than via texture fetches due to texture cache optimization for 2D spatial locality [S4, unofficial].
- **Typo Risk:** Variable naming errors (e.g., `cosntantNCP` vs `constantNCP`) can lead to incorrect constant memory updates [S6, unofficial].

## Subtopics

### [Bindless Textures in CUDA](1980-bindless-textures-in-cuda.md)
Bindless Textures in CUDA is a method for managing and accessing texture data directly from CUDA kernels without the per-frame binding and mapping overhead of OpenGL-CUDA interoperability. It solves the performance bottlenecks and resource limits of traditional OpenGL-CUDA texture sharing, where textures must be bound, mapped to CUDA resources, and unmapped every frame. Unlike [CUDA-OpenGL Interop Surface Write](2228-cuda-opengl-interop-surface-write.md), which focuses on surface buffers, this method addresses texture sampling by allowing a dynamic number of textures to be exposed simultaneously to the kernel at runtime.

### [CUDA-OpenGL Interop Surface Write](2228-cuda-opengl-interop-surface-write.md)
A technique to allow CUDA kernels to write directly to OpenGL 3D textures using CUDA surface objects. It writes directly to an OpenGL texture from a CUDA kernel using CUDA graphics interop, avoiding intermediate device buffers. This method is distinct from [CUDA-OpenGL Interop Texture Copy](1739-cuda-opengl-interop-texture-copy.md) which uses `cudaMemcpy3D` for bulk transfers rather than per-voxel kernel writes.

### PBO to Texture Copy for Display
A display routine that updates the view matrix, renders the volume, clears the screen, copies data from the CUDA PBO to an OpenGL texture, and draws a textured quad [S10, unofficial]. It builds the inverse view matrix by extracting and transposing the model-view matrix components from OpenGL, binds the PBO `pbo` and copies its contents to the 2D texture `tex` using `glTexSubImage2D` with format `GL_RGBA` and type `GL_UNSIGNED_BYTE`, draws a textured quad covering the screen coordinates (0,0) to (1,1) using `glBegin(GL_QUADS)`, and swaps buffers [S10, unofficial].

## Operating characteristics
- **Time Complexity:** The core interop operations (register, map, unmap) are constant time relative to data size but involve API overhead. The rendering loop performance is dominated by the CUDA kernel execution time [S2, S5].
- **Memory Complexity:** Zero-copy sharing eliminates four serial copies and four redundant texture objects compared to high-level libraries like SFML, reducing memory footprint [S2, S5]. However, zero-padding in Fourier methods can increase VRAM usage by a factor of 8 [S4, unofficial].
- **Measured Performance:** On a 512^3 dataset, the pure GPU implementation using CUDA-OpenGL interop outperformed the hybrid (CPU-GPU) one by a factor of 117x for the entire pipeline and 29x for the rendering loop [S4, unofficial]. On NVIDIA GeForce GT 640, the entire pipeline took 693.365 ms; on NVIDIA QUADRO K5000, it took 164.4 ms [S4, unofficial].
- **Scaling:** The method scales with GPU compute power. Trilinear interpolation has almost the same performance as nearest-neighbour for slice sizes ≤ 512^2, but incurs higher performance cost for larger volumes [S4, unofficial].
- **Strengths:** Eliminates CPU bottlenecks from draw calls and API overhead [S2, S5]. Eliminates four serial copies and four redundant texture objects compared to high-level libraries like SFML [S2, S5][S8, unofficial]. Allows greater control over buffer bit depth [S2].
- **Limitations:** Requires manual management of OpenGL and CUDA contexts [S2]. Context switching can be complex [S1][S4, unofficial]. Using SFML for display results in frame latencies >20 ms for 1080p textures, limiting framerate to under 60 fps, whereas PBO/FBO approach reduces this significantly [S2, S5][S8, unofficial].
- **When to Use:** Real-time volume rendering where CPU overhead is a bottleneck, and direct GPU-side computation and display are required [S2, S5].
- **When Not to Use:** Simple applications where CPU-GPU transfer overhead is negligible, or when using higher-level frameworks that abstract away these details and performance is not critical [S2, S5].

## Background
The method originated from the need to overcome CPU bottlenecks in traditional raster engines, where the host issues draw calls faster than the device can complete work [S2]. Early implementations used libraries like SFML, which introduced significant copy overhead [S2, S5]. The shift to CUDA-OpenGL Interoperability allowed for direct GPU-side rendering pipelines, as demonstrated in frameworks like VRCUDA [S6, unofficial] and Fourier Volume Rendering [S1][S4, unofficial].

## Sources
[S1] High Performance GPU-Based Fourier Volume Rendering, https://doi.org/10.1155/2015/590727, fetched 2026-10-27
[S2] Raymarching Distance Fields with CUDA, https://doi.org/10.3390/electronics10222730, fetched 2026-10-26
[S3] pvasbob/HZL-fluidEffectsStudio, https://github.com/pvasbob/HZL-fluidEffectsStudio, fetched 2026-10-27 (unofficial)
[S4] High Performance GPU-Based Fourier Volume Rendering - PMC, https://pmc.ncbi.nlm.nih.gov/articles/PMC4381991/, fetched 2026-10-27 (unofficial)
[S5] Raymarching Distance Fields with CUDA, https://www.mdpi.com/2079-9292/10/22/2730, fetched 2026-10-27
[S6] franjaviersans/VRCUDA, https://github.com/franjaviersans/VRCUDA, fetched 2026-10-27 (unofficial)
[S7] Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, https://github.com/Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, fetched 2026-10-27 (unofficial)
[S8] (PDF) Raymarching Distance Fields with CUDA - ResearchGate, https://www.researchgate.net/publication/356081826_Raymarching_Distance_Fields_with_CUDA, fetched 2026-10-27 (unofficial)
[S9] Realtime visualization of 3D vector fields using CUDA | Marek's blog, https://marekfiser.com/projects/realtime-visualization-of-3D-vector-fields-using-CUDA/, fetched 2026-10-28 (unofficial)
[S10] RubbishKing0624/VolumeRendering, https://github.com/RubbishKing0624/VolumeRendering, fetched 2026-10-26 (unofficial)
