# CUDA-OpenGL Interop Surface Write

Also called: Direct GL Texture Write from CUDA, Surface Write via CUDA Interop

## Overview
CUDA-OpenGL Interop Surface Write writes directly to an OpenGL texture from a CUDA kernel using CUDA graphics interop, avoiding intermediate device buffers [S1, unofficial]. It sets itself apart from standard interop methods by allowing registration with `cudaGraphicsRegisterFlagsSurfaceLoadStore` for read/write access or `cudaGraphicsRegisterFlagsWriteDiscard` for write-only access [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: An OpenGL texture ID (created as RGBA8), CUDA device memory containing scene data (camera, volume, transfer function), and image dimensions ($W, H$) [S1, unofficial].
- Output: The OpenGL texture updated with the rendered image pixels [S1, unofficial].

**Data Structures:**
- `cudaGraphicsResource_t`: Handle to the registered OpenGL resource [G].
- `cudaArray_t`: Mapped array representation of the OpenGL texture in CUDA memory [G].
- `cudaSurfaceObject_t`: Surface object bound to the `cudaArray_t` for writing [G].

**Steps:**
1. **Create GL Texture:** Generate an OpenGL texture with format `GL_RGBA8` and dimensions $W \times H$ [S1, unofficial].
2. **Register Resource:** Register the GL texture with CUDA using `cudaGraphicsGLRegisterImage` [G]. The registration flags depend on the access mode: use `cudaGraphicsRegisterFlagsSurfaceLoadStore` if `useSurfaceWrite_` is true, else use `cudaGraphicsRegisterFlagsWriteDiscard` [S1, unofficial].
3. **Map Resource:** Before the kernel launch, map the CUDA resource to make it accessible to CUDA kernels using `cudaGraphicsMapResources` [G].
4. **Get Mapped Array:** Retrieve the `cudaArray_t` handle for the mapped resource using `cudaGraphicsSubResourceGetMappedArray` [G].
5. **Launch Kernel:** Launch the CUDA kernel passing the mapped array or surface object [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cuda_opengl_surface_write(gl_texture_id, width, height, use_surface_write):
    # 1. Create GL RGBA8 texture (assumed done externally)
    # 2. Register GL texture with CUDA
    flags = cudaGraphicsRegisterFlagsSurfaceLoadStore if use_surface_write else cudaGraphicsRegisterFlagsWriteDiscard
    cuda_resource = cudaGraphicsGLRegisterImage(gl_texture_id, GL_TEXTURE_2D, flags)

    # 3. Map CUDA resource to get cudaArray_t
    cudaGraphicsMapResources(1, cuda_resource)
    dst_array = cudaGraphicsSubResourceGetMappedArray(cuda_resource, 0, 0)

    # 4. Launch kernel
    if use_surface_write:
        surface_obj = cudaCreateSurfaceObject(dst_array)
        kernel_ray_hit(scene_data, surface_obj, width, height)
        cudaDestroySurfaceObject(surface_obj)
    else:
        kernel_ray_hit(scene_data, dst_array, width, height)

    # 5. Unmap resource (standard interop cleanup)
    cudaGraphicsUnmapResources(1, cuda_resource)
```

**Parameters:**
- `width`, `height`: Image dimensions [S1, unofficial].
- `cudaGraphicsRegisterFlags`: `SurfaceLoadStore` or `WriteDiscard` [S1, unofficial].

**Edge Cases and Pitfalls:**
- The GL texture format must match the data type written by the kernel (e.g., `uchar4` for `GL_RGBA8`) [S1, unofficial].
- The GL texture must be bound and valid at the time of registration [G].

## Subtopics

### CUDA-OpenGL Interop Texture Copy
Copies a 3D volume of float data from device memory into a registered OpenGL 3D texture via CUDA interop, mapping the resource, performing a 3D memory copy, and unmapping [S2, unofficial].
- **Steps:**
  1. Map resource: Call `cudaGraphicsMapResources(1, &m_resource)` [S2, unofficial].
  2. Get mapped array: Call `cudaGraphicsSubResourceGetMappedArray(&array, m_resource, 0, 0)` [S2, unofficial].
  3. Configure copy parameters: Set `srcPtr` with `make_cudaPitchedPtr(sourcePointer, width*sizeof(float), width, height)`, `dstArray=array`, `extent=make_extent(width, height, depth)`, `kind=cudaMemcpyDeviceToDevice` [S2, unofficial].
  4. Execute copy: Call `cudaMemcpy3D(&copy)` [S2, unofficial].
  5. Unmap resource: Call `cudaGraphicsUnmapResources(1, &m_resource)` [S2, unofficial].
- **Pitfall:** Source pointer must not be null and resource must be registered; throws `std::logic_error` if either condition fails [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Eliminates the final CPU-GPU transfer bottleneck for the rendered image [S1, unofficial].
- **Limitations:** Not given in the sources.
- **When to use:** Not given in the sources.
- **When not to use:** Not given in the sources.

## Background
Originated from the need to integrate CUDA compute capabilities with OpenGL rendering pipelines efficiently. The technique is demonstrated in the cuVRender project, which implements a CUDA-accelerated volume rendering engine [S1, unofficial].

## Sources
[S1] Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, https://github.com/Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, fetched 2026-10-27 (unofficial)
[S2] pvasbob/HZL-fluidEffectsStudio, https://github.com/pvasbob/HZL-fluidEffectsStudio, fetched 2026-10-27 (unofficial)
