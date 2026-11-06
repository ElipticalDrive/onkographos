# CUDA-OpenGL Interoperability

Also called: CUDA-GL Interop, CUDA-to-OpenGL PBO write, OpenGL-CUDA Interoperability via Pixel Buffer Object (PBO), PBO-based rendering

## Overview
CUDA-OpenGL Interoperability is a mechanism that allows CUDA kernels to read from and write to OpenGL resources (textures, buffers) directly in GPU memory, eliminating CPU-mediated data transfers [S2][S4, unofficial][S5][S6, S7, unofficial]. It solves the bottleneck of copying rendered volume data or simulation results from CUDA device memory back to system memory for display [S2]. It sets itself apart from standard interop methods by supporting both read-only access (via mapped arrays/texture objects) and write access (via surface objects or direct buffer pointers), enabling zero-copy rendering pipelines [S1][S3, S4, unofficial][S5][S6, S7, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** OpenGL resource IDs (Texture ID, Buffer ID), resource dimensions (width, height, depth), and CUDA graphics resource flags [S1][S3, S6, unofficial].
*   **Outputs:** CUDA graphics resource handle (`cudaGraphicsResource_t`), mapped CUDA array (`cudaArray_t`), or device pointer (`void*`) [S1][S3, unofficial][S5].

**Data Structures:**
*   `cudaGraphicsResource_t`: Opaque handle returned by registration functions [S1][S3, S6, unofficial].
*   `cudaArray_t`: Device memory structure representing the mapped OpenGL texture [S1][S3, S6, unofficial].
*   `cudaResourceDesc`: Structure describing the resource type (Array, Buffer) and its properties [S1][S3, S6, unofficial].
*   `cudaSurfaceObject_t`: Object allowing atomic writes to a mapped array [S1][S3, S6, unofficial].

**Numbered Steps:**

1.  **Resource Registration:**
    *   For Textures: Call `cudaGraphicsGLRegisterImage` with the OpenGL texture ID, target (`GL_TEXTURE_2D` or `GL_TEXTURE_3D`), and flags (`cudaGraphicsRegisterFlagsSurfaceLoadStore` for write, `cudaGraphicsMapFlagsReadOnly` for read) [S1][S3, S4, S6, unofficial].
    *   For Buffers (VBO/PBO): Call `cudaGraphicsGLRegisterBuffer` with the buffer ID and flags (`cudaGraphicsMapFlagsNone`) [S5][S9, unofficial].
    *   *Pitfall:* Ensure the OpenGL context is current before registration [S2].

2.  **Resource Mapping (Per Frame/Per Kernel Launch):**
    *   Call `cudaGraphicsMapResources` with the resource handle count and array of handles to gain CUDA access [S1][S3, S4, unofficial][S5][S6, unofficial].
    *   *Pitfall:* Accessing the mapped resource with OpenGL before unmapping causes program failure [S5][S9, unofficial].

3.  **Resource Acquisition:**
    *   For Textures: Call `cudaGraphicsSubResourceGetMappedArray` to retrieve the `cudaArray_t` [S1][S3, S4, S6, unofficial].
    *   For Buffers: Call `cudaGraphicsResourceGetMappedPointer` to retrieve the device pointer (`void**`) [S5][S9, unofficial].

4.  **Binding (Optional but Recommended for Textures):**
    *   For Read Access: Bind the `cudaArray_t` to a CUDA texture object using `cudaBindTextureToArray` [S3, S6, unofficial].
    *   For Write Access: Initialize `cudaResourceDesc` with `resType=cudaResourceTypeArray` and `res.array.array=array`, then call `cudaCreateSurfaceObject` to create a `cudaSurfaceObject_t` [S1][S3, S6, unofficial]. Alternatively, bind the array to a surface using `cudaBindSurfaceToArray` [S3, S6, unofficial].

5.  **Kernel Execution:**
    *   Launch the CUDA kernel.
    *   If using Surface: Pass the `cudaSurfaceObject_t` to the kernel. Use surface store or atomic functions for writes [S1][S3, S6, unofficial].
    *   If using Pointer (VBO): Pass the device pointer obtained in Step 3. The kernel writes directly to the VBO memory [S5][S9, unofficial].
    *   If using Texture Array (Read-only): Bind the array to a CUDA texture object and use texture fetch functions for reads [S3, S6, unofficial].

6.  **Unmapping:**
    *   Call `cudaGraphicsUnmapResources` to release CUDA access, allowing OpenGL to use the resource again [S1][S3, S4, unofficial][S5][S6, unofficial].

7.  **Cleanup (Optional):**
    *   Destroy surface objects if created: `cudaDestroySurfaceObject` [S1][S3, unofficial].
    *   Unregister resources on shutdown: `cudaGraphicsUnregisterResource` [S1][S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Standard CUDA Runtime API definitions (provided by cuda_runtime.h)
def cudaGraphicsGLRegisterImage(texture_id, target, flags): ...
def cudaGraphicsGLRegisterBuffer(buffer_id, flags): ...
def cudaGraphicsMapResources(count, handles): ...
def cudaGraphicsSubResourceGetMappedArray(resource, level, miplevel): ...
def cudaGraphicsResourceGetMappedPointer(resource): ...
def cudaBindTextureToArray(texture, array): ...
def cudaCreateSurfaceObject(desc): ...
def cudaBindSurfaceToArray(surface, array): ...
def cudaGraphicsUnmapResources(count, handles): ...
def cudaDestroySurfaceObject(surface): ...
def cudaGraphicsUnregisterResource(resource): ...

# Constants
cudaGraphicsRegisterFlagsSurfaceLoadStore = 0x01
cudaGraphicsRegisterFlagsReadOnly = 0x02
cudaGraphicsMapFlagsNone = 0x00
cudaResourceTypeArray = 0x00
GL_TEXTURE_2D = 0x0DE1
GL_TEXTURE_3D = 0x806F

# Initialization (One-time)
def init_opengl_texture_interop(gl_texture_id, width, height, depth, is_writable=False):
    flags = cudaGraphicsRegisterFlagsSurfaceLoadStore if is_writable else cudaGraphicsRegisterFlagsReadOnly
    resource_handle = cudaGraphicsGLRegisterImage(gl_texture_id, GL_TEXTURE_3D, flags)
    surface_handle = 0
    return resource_handle, surface_handle

def init_opengl_buffer_interop(gl_vbo_id, size_bytes):
    resource_handle = cudaGraphicsGLRegisterBuffer(gl_vbo_id, cudaGraphicsMapFlagsNone)
    return resource_handle

# Per-Frame Execution
def render_frame(resource_handle, surface_handle, gl_vbo_handle=None, width=1920, height=1080):
    cudaGraphicsMapResources(1, resource_handle)
    
    cuda_array = None
    device_ptr = None
    
    if gl_vbo_handle is None:
        cuda_array = cudaGraphicsSubResourceGetMappedArray(resource_handle, 0, 0)
        
        if surface_handle == 0:
            res_desc = cudaResourceDesc()
            res_desc.resType = cudaResourceTypeArray
            res_desc.res.array.array = cuda_array
            surface_handle = cudaCreateSurfaceObject(res_desc)
        
        # launch_kernel<<<blocks, threads>>>(surface_handle, width, height)
    else:
        device_ptr, bytes_count = cudaGraphicsResourceGetMappedPointer(resource_handle)
        # launch_kernel<<<blocks, threads>>>(device_ptr, width, height)
    
    cudaGraphicsUnmapResources(1, resource_handle)
    return surface_handle

# Cleanup
def cleanup(resource_handle, surface_handle):
    if surface_handle != 0:
        cudaDestroySurfaceObject(surface_handle)
    cudaGraphicsUnregisterResource(resource_handle)
```

**Parameters:**
*   `block_dim`: Typically `dim3(16, 16, 1)` for 2D image writing [S3, unofficial].
*   `grid_dim`: Calculated as `(Width + block_dim.x) / block_dim.x` for x, `(Height + block_dim.y) / block_dim.y` for y, and 1 for z [S3, unofficial].
*   `constantWidth`, `constantHeight`: CUDA constants storing viewport dimensions, updated via `cudaMemcpyToSymbol` [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Context Current:** OpenGL context must be current when registering or mapping resources [S2].
*   **Surface Creation Failure:** If `cudaGraphicsSubResourceGetMappedArray` fails, the surface object is not created, preventing kernel launch; check `cudaSuccess` before creating the surface [S1][S3, unofficial].
*   **Typo in Constants:** Ensure variable names match exactly when using `cudaMemcpyToSymbol` (e.g., `constantNCP` vs `cosntantNCP`) [S3, unofficial].
*   **Ray-Box Intersection:** If `NOT_RAY_BOX` is defined, front/back hit positions are computed via OpenGL geometry shaders and stored in textures; otherwise, the inverse view matrix is passed to CUDA for GPU-side intersection [S3, unofficial].

## Subtopics

### [CUDA-OpenGL Interop Surface Write](2228-cuda-opengl-interop-surface-write.md)
CUDA-OpenGL Interop Surface Write writes directly to an OpenGL texture from a CUDA kernel using CUDA graphics interop, avoiding intermediate device buffers. It sets itself apart from standard interop methods by allowing registration with `cudaGraphicsRegisterFlagsSurfaceLoadStore` for read/write access or `cudaGraphicsRegisterFlagsWriteDiscard` for write-only access.

### [Bindless Textures in CUDA](1980-bindless-textures-in-cuda.md)
Bindless Textures in CUDA is an object-oriented approach using the CUDA API to upload textures into VRAM and expose them via texture object descriptors, eliminating the need for per-frame binding and mapping required by OpenGL-CUDA interoperability for textures. It sets itself apart by allowing a dynamic number of textures specified at runtime, unlike the OpenGL mapping method which requires per-frame binding/unbinding.

## Operating characteristics
*   **Time Complexity:** O(1) for registration/unregistration (one-time) [S1][S3, unofficial]. O(1) for map/unmap per frame [S1][S3, unofficial]. Kernel execution time depends on the rendering algorithm (e.g., raymarching) [S2][S3, unofficial].
*   **Memory Complexity:** Zero-copy; no additional CPU memory allocation for the rendered image or texture data [S2][S4, unofficial].
*   **Measured Performance:** Eliminates four serial copies and four redundant texture objects compared to SFML-based approaches [S2, S5]. Reduces frame latency from >20 ms (limiting framerate to under 60 fps for 1080p) to significantly lower values [S2, S5][S8, unofficial]. On a 512^3 dataset, volume uploading takes 100 ms (GPU 1) or 40 ms (GPU 2); 3D FFT takes 442.004 ms or 90.53 ms; entire pipeline takes 693.365 ms or 164.4 ms [S4, unofficial]. Pure GPU implementation outperforms hybrid by 117x for the pipeline and 29x for the rendering loop [S4, unofficial].
*   **Strengths:** Eliminates CPU-GPU data transfer overhead; allows direct GPU-side rendering; supports atomic writes via surfaces [S2, S5][S6, unofficial].
*   **Limitations:** Requires OpenGL context management; binding/unmapping overhead for textures; limited simultaneous texture count in standard OpenGL interop [S2][S3, unofficial].
*   **When to Use:** Real-time volume rendering, GPU-based simulations requiring visual feedback, avoiding CPU bottlenecks [S2, S5].
*   **When Not to Use:** When CPU-side post-processing is required, or when using libraries that abstract away OpenGL/CUDA interop (e.g., SFML) where performance is not critical [S2, S5].

## Background
CUDA-OpenGL Interoperability was introduced with CUDA 2.0 [S2]. It has been widely used in volume rendering engines (e.g., VRCUDA, cuVRender) and fluid simulation visualizations (e.g., HZL-fluidEffectsStudio) to achieve real-time performance by bypassing the CPU [S1][S3, S4, unofficial][S5][S6, S7, unofficial].

## Sources
[S1] pvasbob/HZL-fluidEffectsStudio, https://github.com/pvasbob/HZL-fluidEffectsStudio, fetched 2026-10-27
[S2] Raymarching Distance Fields with CUDA, https://www.mdpi.com/2079-9292/10/22/2730, fetched 2026-10-27
[S3] franjaviersans/VRCUDA, https://github.com/franjaviersans/VRCUDA, fetched 2026-10-27 (unofficial)
[S4] Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, https://github.com/Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, fetched 2026-10-27 (unofficial)
[S5] Realtime visualization of 3D vector fields using CUDA | Marek's blog, https://marekfiser.com/projects/realtime-visualization-of-3D-vector-fields-using-CUDA/, fetched 2026-10-28
