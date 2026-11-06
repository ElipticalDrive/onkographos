# CUDA-OpenGL Buffer Interoperability

Also called: CUDA-OpenGL buffer sharing, CUDA-OpenGL Interop for Buffers, OpenGL/CUDA Buffer Interoperability, Shared Buffer Objects

## Overview
This method enables zero-copy data exchange between CUDA compute kernels and the OpenGL rendering pipeline by mapping OpenGL Buffer Objects (BOs) directly into CUDA memory space [S1, unofficial][S2][S3, unofficial]. It solves the performance bottleneck of transferring GPU-generated geometry (e.g., isosurface vertices, normals) or particle data through the CPU, allowing CUDA kernels to write directly into buffers readable by OpenGL vertex shaders [S2]. Unlike the Unreal Engine backend which uses host-side staging interfaces, this method provides a true zero-copy path for standalone renderers [S2]. It is distinct from general CUDA-OpenGL Interoperability [1738-cuda-opl-interoperability.md] by focusing specifically on the lifecycle management of Buffer Objects for vertex/fragment data rather than general resource sharing.

## How it works
**Inputs:** OpenGL Vertex Buffer Object (VBO) or Element Buffer Object (EBO) handle; CUDA context.
**Outputs:** Mapped CUDA device pointer; updated buffer contents visible to OpenGL.

**Data Structures:**
*   **OpenGL Buffer Object:** Allocated with `GL_DYNAMIC_DRAW` usage hint to indicate frequent updates [S1, unofficial].
*   **CUDA Graphics Resource:** A handle returned by `cudaGraphicsRegisterBuffer` representing the registered VBO/EBO [S2].

**Steps:**
1.  **Allocation:** Create the OpenGL buffer using `glGenBuffers` and `glBufferData` with the `GL_DYNAMIC_DRAW` flag [S1, unofficial].
2.  **Registration:** Register the OpenGL buffer handle with the CUDA runtime using `cudaGraphicsGLRegisterBuffer`. The registration flags should include `cudaGraphicsRegisterFlagsWriteDiscard` if the kernel will overwrite the entire buffer content, allowing CUDA to optimize memory access [S1, unofficial].
3.  **Mapping (Integration Phase):** Before the CUDA kernel executes, the registered resource must be mapped using `cudaGraphicsMapResources`. This operation:
    *   Synchronizes the CUDA and OpenGL contexts [S3, unofficial].
    *   Returns a device pointer to the buffer's memory space [S3, unofficial].
    *   Makes the buffer inaccessible to OpenGL rendering and host modification during the mapping [S3, unofficial].
4.  **Computation:** Launch the CUDA kernel. The kernel writes data (e.g., vertex positions, normals) directly to the mapped device pointer [S2].
5.  **Unmapping (Rendering Phase):** After the kernel completes, call `cudaGraphicsUnmapResources`. This releases the buffer from CUDA access [S2].
6.  **Rendering:** Bind the buffer as a standard vertex attribute in OpenGL and perform rasterization [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization (once)
def init_interop(gl_buffer_id):
    # Register the OpenGL buffer with CUDA
    cuda_resource = cudaGraphicsGLRegisterBuffer(gl_buffer_id, cudaGraphicsRegisterFlagsWriteDiscard)
    return cuda_resource

def render_frame(cuda_resource, gl_buffer_id, kernel_args):
    # 1. Map resource for CUDA access
    # This implicitly synchronizes CUDA and OpenGL threads [S3, unofficial]
    # While mapped, the buffer cannot be used for rendering [S3, unofficial]
    mapped_ptr = cudaGraphicsMapResources(cuda_resource)
    
    # 2. Execute CUDA Kernel
    # Kernel writes directly to mapped_ptr
    launch_cuda_kernel(mapped_ptr, kernel_args)
    
    # 3. Unmap resource to release CUDA access
    cudaGraphicsUnmapResources(cuda_resource)
    
    # 4. Render with OpenGL
    glBindBuffer(GL_ARRAY_BUFFER, gl_buffer_id)
    # ... set vertex attributes ...
    glDrawArrays(...)
```

**Parameters:**
*   `cudaGraphicsRegisterFlagsWriteDiscard`: Recommended if the kernel overwrites all data, allowing CUDA to discard previous contents efficiently [S1, unofficial].
*   `GL_DYNAMIC_DRAW`: OpenGL usage hint indicating the buffer contents will be changed frequently [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Synchronization:** Mapping a buffer implicitly synchronizes CUDA and OpenGL threads [S3, unofficial]. Failure to unmap before OpenGL access causes errors or hangs.
*   **Resident Buffers:** If using the Shader Buffer Load extension, a Buffer Object can be declared "resident" for shader access. A resident BO is unavailable for CUDA or host modification and must be made non-resident before CUDA mapping [S3, unofficial].
*   **Engine Specifics:** This zero-copy mechanism applies to standalone OpenGL renderers. In engines like Unreal Engine, geometry is typically transferred via host-side staging interfaces (e.g., `ProceduralMeshComponent`) rather than direct CUDA-OpenGL sharing [S2].

## Operating characteristics
*   **Time Complexity:** O(1) for the mapping/unmapping overhead relative to data size, but incurs a synchronization penalty due to context switching between OpenGL and CUDA [S3, unofficial].
*   **Memory Complexity:** Zero additional memory copies; the buffer resides in GPU VRAM [S2].
*   **Performance:** Eliminates PCIe bus transfers between GPU and CPU [S2]. Performance is limited by the synchronization latency when alternating between CUDA and OpenGL access [S3, unofficial].
*   **Strengths:** Ideal for real-time visualization of dynamically generated geometry (isosurfaces, particles) where data is generated and consumed on the GPU [S2].
*   **Limitations:** Not applicable to all rendering engines (e.g., Unreal Engine uses staging) [S2]. Requires careful management of resource residency and mapping states to avoid context conflicts [S3, unofficial].

## Background
The technique was formalized in the CUDA Toolkit documentation and widely adopted in scientific visualization for GPU-based geometry generation [S2]. It allows for the direct transfer of isosurface extraction results to the rendering pipeline without CPU intervention [S2]. The concept of Buffer Object life cycles and resident buffers for shader access was detailed in academic works on real-time volume rendering and smoke simulation [S3, unofficial].

## Sources
[S1] keckj/cupmb, https://github.com/keckj/cupmb, fetched 2026-10-28 (unofficial)
[S2] Visual tools for 3D plasma simulation: GPU-based isosurface extraction ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC13674866/, fetched 2026-10-30
[S3] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
