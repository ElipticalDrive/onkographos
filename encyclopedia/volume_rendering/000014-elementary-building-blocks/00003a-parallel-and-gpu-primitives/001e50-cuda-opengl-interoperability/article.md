# CUDA-OpenGL Interoperability

Also called: CUDA-GL Interop, CUDA-to-OpenGL PBO write, OpenGL-CUDA Interoperability via Pixel Buffer Object (PBO), PBO-based rendering

## Overview
CUDA-OpenGL Interoperability is a method for sharing GPU memory between CUDA computational kernels and the OpenGL rendering pipeline without CPU-mediated data transfers [S2][S4, unofficial][S5]. It solves the bottleneck where the CPU issues draw calls or copies data between host and device memory, which limits frame rates in real-time volume rendering [S2, S5]. Unlike traditional raster engines that rely on draw calls and shader binding, this method allows CUDA kernels to write directly to OpenGL resources (such as Pixel Buffer Objects or 3D textures) or read from them via mapped CUDA arrays, enabling zero-copy data sharing between simulation and rendering [S3, S4, unofficial][S5].

## How it works
The method establishes a bridge between the CUDA and OpenGL contexts using the `cudaGraphicsRegister` API family. It supports two primary resource types: Pixel Buffer Objects (PBOs) for image output and OpenGL Textures (2D/3D) for data input or output via surface objects.

### Inputs and Outputs
- **Inputs:** OpenGL resource IDs (texture IDs, buffer IDs), OpenGL context handle, CUDA device ID.
- **Outputs:** CUDA graphics resource handles (`cudaGraphicsResource_t`), CUDA array views (`cudaArray_t`), CUDA surface objects (`cudaSurfaceObject_t`), or device pointers (`void*`).

### Data Structures and Layouts

**1. CUDA Graphics Resource Handle**
A opaque handle returned by registration functions, used to map/unmap the resource.
- Type: `cudaGraphicsResource_t`

**2. CUDA Array View**
A linear memory structure representing the OpenGL texture data, accessible via texture/surface references.
- Type: `cudaArray_t`

**3. CUDA Surface Object**
A writable view of a CUDA array, allowing direct per-voxel/pixel writes from kernels.
- Type: `cudaSurfaceObject_t`

**4. CudaGlTexture3D Class Layout (Example Implementation)**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct CudaGlTexture3D {
    cudaGraphicsResource* m_resource; // Handle to the registered OpenGL resource
    int m_width;                      // Texture width
    int m_height;                     // Texture height
    int m_depth;                      // Texture depth (for 3D textures)
};
```
[S3, unofficial]

### Numbered Steps

The general workflow involves registration, mapping, computation, unmapping, and rendering.

#### Phase 1: Registration (Initialization)
1. **Create OpenGL Resource:**
   - For PBO: Generate a buffer using `glGenBuffers`, bind to `GL_PIXEL_UNPACK_BUFFER`, and allocate storage with `glBufferData`.
   - For Texture: Generate a texture using `glGenTextures`, bind to `GL_TEXTURE_3D` (or `GL_TEXTURE_2D`), and set parameters.
   - For VBO: Generate a buffer using `glGenBuffers`, bind to `GL_ARRAY_BUFFER`, and allocate storage.
   [S2, S5][S9, unofficial]

2. **Register with CUDA:**
   - Call `cudaGraphicsGLRegisterImage` for textures or `cudaGraphicsGLRegisterBuffer` for PBOs/VBOs.
   - Specify the target (e.g., `GL_TEXTURE_3D`, `GL_PIXEL_UNPACK_BUFFER`).
   - Set flags:
     - `cudaGraphicsRegisterFlagsSurfaceLoadStore`: Allows CUDA kernels to write to the resource (e.g., for output textures).
     - `cudaGraphicsMapFlagsReadOnly`: If the resource is only read by CUDA (e.g., transfer function textures).
     - `cudaGraphicsMapFlagsNone`: Default for buffers.
   [S3, S6, S7, S9, unofficial]

   *Example Code:*
   ```cpp
   cudaGraphicsResource_t cudaResource;
   // For a 3D texture
   cudaGraphicsGLRegisterImage(&cudaResource, textureId, GL_TEXTURE_3D, cudaGraphicsRegisterFlagsSurfaceLoadStore);
   // For a PBO
   cudaGraphicsGLRegisterBuffer(&cudaResource, pboId, cudaGraphicsMapFlagsNone);
   ```
   [S3, S6, S7, S9, unofficial]

#### Phase 2: Mapping (Per-Frame)
1. **Map Resource:**
   - Call `cudaGraphicsMapResources(count, &resource, stream)` to gain CUDA access.
   - This synchronizes the resource between OpenGL and CUDA contexts.
   [S3, S6, S7, S9, unofficial]

2. **Get Mapped View:**
   - **For Textures:** Call `cudaGraphicsSubResourceGetMappedArray(&array, resource, 0, 0)` to retrieve the `cudaArray_t`.
     - Bind to CUDA Texture Object: `cudaBindTextureToArray(texRef, array, channelDesc)`.
     - Bind to CUDA Surface Object: `cudaBindSurfaceToArray(surf, array)`.
   [S3, S6, S7, unofficial]
   - **For Buffers (PBO/VBO):** Call `cudaGraphicsResourceGetMappedPointer(&ptr, &bytes, resource)` to retrieve the raw device pointer.
   [S9, unofficial]

#### Phase 3: Computation
1. **Launch CUDA Kernel:**
   - The kernel reads from mapped textures (if input) or writes to mapped arrays/pointers (if output).
   - **PBO Write:** Kernel writes pixel data directly to the PBO memory.
   - **Texture Surface Write:** Kernel writes voxel data directly to the 3D texture via surface object.
   - **VBO Write:** Kernel writes vertex/attribute data directly to the VBO memory.
   [S2, S5][S7, S9, unofficial]

2. **Synchronize:**
   - Call `cudaDeviceSynchronize()` or `cudaStreamSynchronize(stream)` to ensure kernel completion before OpenGL access.
   [S3, S7, unofficial]

#### Phase 4: Unmapping and Rendering
1. **Unmap Resource:**
   - Call `cudaGraphicsUnmapResources(count, &resource, stream)` to release CUDA access.
   - This allows OpenGL to use the resource again.
   [S3, S6, S7, S9, unofficial]

2. **Display/Render:**
   - **PBO Method:**
     - Option A: Bind PBO to `GL_PIXEL_UNPACK_BUFFER` and draw a fullscreen quad using a trivial shader.
     - Option B: Use an explicit Frame Buffer Object (FBO) where the PBO is attached or used to build the frame buffer, avoiding rasterization hardware for display.
   [S2, S5]
   - **Texture Method:**
     - Bind the OpenGL texture to a texture unit.
     - Draw a textured quad (or use the texture in a subsequent render pass).
   [S6, S10, unofficial]
   - **VBO Method:**
     - Bind VBOs, set vertex pointers, and call `glDrawArrays` or `glDrawElements`.
   [S9, unofficial]

### Pseudocode: PBO-Based Rendering Loop

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization (once)
def init_cuda_gl_interop(pbo_id):
    # 1. Register PBO with CUDA
    cuda_resource = cudaGraphicsGLRegisterBuffer(pbo_id, cudaGraphicsMapFlagsNone)
    return cuda_resource

def render_frame(cuda_resource, width, height, kernel_func):
    # 1. Map the PBO into CUDA address space
    cudaGraphicsMapResources(1, cuda_resource, 0)
    
    # 2. Get device pointer to the PBO memory
    device_ptr, bytes_count = cudaGraphicsResourceGetMappedPointer(cuda_resource)
    
    # 3. Launch CUDA kernel to write image data to PBO
    # Kernel signature: __global__ void render_kernel(unsigned char* output, int width, int height)
    block_dim = dim3(16, 16, 1)
    grid_dim = dim3((width + 15) // 16, (height + 15) // 16, 1)
    kernel_func<<<grid_dim, block_dim>>>(device_ptr, width, height)
    
    # 4. Synchronize CUDA execution
    cudaDeviceSynchronize()
    
    # 5. Unmap the PBO to release CUDA access
    cudaGraphicsUnmapResources(1, cuda_resource, 0)
    
    # 6. OpenGL Rendering (Display)
    # Option: Bind PBO and draw quad, or use FBO approach
    glActiveTexture(GL_TEXTURE0)
    glBindTexture(GL_TEXTURE_2D, display_texture_id)
    glTexSubImage2D(GL_TEXTURE_2D, 0, 0, 0, width, height, GL_RGBA, GL_UNSIGNED_BYTE, device_ptr)
    draw_textured_quad()
    swap_buffers()
```
[G] *Note: The `cudaGraphicsGLRegisterBuffer` and related functions are CUDA Runtime API calls. The OpenGL calls (`glTexSubImage2D`, `draw_textured_quad`) are standard OpenGL. The kernel launch syntax is CUDA C++.*

### Parameters and Typical Values
- **Block Dimension:** Typically `dim3(16, 16, 1)` for 2D image rendering [S6, unofficial].
- **Grid Dimension:** Calculated as `(Width + block_dim.x) / block_dim.x` for x, `(Height + block_dim.y) / block_dim.y` for y [S6, unofficial].
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
Bindless Textures in CUDA is a method for managing and accessing texture data directly from CUDA kernels without the overhead of OpenGL-CUDA interoperability binding and mapping cycles. It solves the performance bottlenecks and resource limitations inherent in traditional OpenGL-CUDA texture sharing, where textures must be bound, mapped to CUDA resources, and unmapped every frame. Unlike [CUDA-OpenGL Interop Surface Write](2228-cuda-opengl-interop-surface-write.md) which focuses on surface buffers, this method specifically addresses texture sampling, allowing a dynamic number of textures to be exposed simultaneously to the kernel at runtime.

### [CUDA-OpenGL Interop Surface Write](2228-cuda-opengl-interop-surface-write.md)
A technique to allow CUDA kernels to write directly to OpenGL 3D textures using CUDA surface objects. It writes directly to an OpenGL texture from a CUDA kernel using CUDA graphics interop, avoiding intermediate device buffers. This method is distinct from [CUDA-OpenGL Interop Texture Copy](1739-cuda-opengl-interop-texture-copy.md) which uses `cudaMemcpy3D` for bulk transfers rather than per-voxel kernel writes.

### PBO to Texture Copy for Display
A display routine that updates the view matrix, renders the volume, clears the screen, copies data from the CUDA PBO to an OpenGL texture, and draws a textured quad [S10, unofficial]. It builds the inverse view matrix by extracting and transposing the model-view matrix components from OpenGL, binds the PBO `pbo` and copies its contents to the 2D texture `tex` using `glTexSubImage2D` with format `GL_RGBA` and type `GL_UNSIGNED_BYTE`, draws a textured quad covering the screen coordinates (0,0) to (1,1) using `glBegin(GL_QUADS)`, and swaps buffers [S10, unofficial].

## Operating characteristics
- **Time Complexity:** The core interop operations (register, map, unmap) are constant time relative to data size but involve API overhead. The rendering loop performance is dominated by the CUDA kernel execution time.
- **Memory Complexity:** Zero-copy sharing eliminates the need for double-buffering in system memory, reducing memory footprint by avoiding host-side copies [S2, S5]. However, zero-padding in Fourier methods can increase VRAM usage by a factor of 8 [S4, unofficial].
- **Measured Performance:** On a 512^3 dataset, the pure GPU implementation using CUDA-OpenGL interop outperformed the hybrid (CPU-GPU) one by a factor of 117x for the entire pipeline and 29x for the rendering loop [S4, unofficial]. On NVIDIA GeForce GT 640, the entire pipeline took 693.365 ms; on NVIDIA QUADRO K5000, it took 164.4 ms [S4, unofficial].
- **Scaling:** The method scales with GPU compute power. Trilinear interpolation has almost the same performance as nearest-neighbour for slice sizes <= 512^2, but incurs higher performance cost for larger volumes [S4, unofficial].
- **Strengths:** Eliminates CPU bottlenecks from draw calls and API overhead [S2, S5]. Eliminates four serial copies and four redundant texture objects compared to high-level libraries like SFML [S2, S5][S8, unofficial]. Allows greater control over buffer bit depth [S2].
- **Limitations:** Requires manual management of OpenGL and CUDA contexts [S2]. Context switching can be complex [S1][S4, unofficial]. Using SFML for display results in frame latencies >20 ms for 1080p textures, limiting framerate to under 60 fps, whereas PBO/FBO approach reduces this significantly [S2, S5][S8, unofficial].
- **When to Use:** Real-time volume rendering where CPU overhead is a bottleneck, and direct GPU-side computation and display are required.
- **When Not to Use:** Simple applications where CPU-GPU transfer overhead is negligible, or when using higher-level frameworks that abstract away these details and performance is not critical.

## Background
The method originated from the need to overcome CPU bottlenecks in traditional raster engines, where the host issues draw calls faster than the device can complete work [S2]. Early implementations used libraries like SFML, which introduced significant copy overhead [S2, S5]. The shift to CUDA-OpenGL Interoperability allowed for direct GPU-side rendering pipelines, as demonstrated in frameworks like VRCUDA [S6, unofficial] and Fourier Volume Rendering [S1][S4, unofficial]. It is widely used in scientific visualization, medical imaging, and real-time volume rendering engines where performance is critical [S1][S4, S6, unofficial].

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
