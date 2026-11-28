# CUDA-OpenGL Interop Surface Write

Also called: Direct GL Texture Write from CUDA, Surface Write via CUDA Interop

## Overview
A technique to allow CUDA kernels to write directly to OpenGL 3D textures using CUDA surface objects [S1, unofficial]. It writes directly to an OpenGL texture from a CUDA kernel using CUDA graphics interop, avoiding intermediate device buffers [S2, unofficial]. This method is distinct from [CUDA-OpenGL Interop Texture Copy](1739-cuda-opengl-interop-texture-copy.md), which uses `cudaMemcpy3D` for bulk transfers rather than per-voxel kernel writes [S2, unofficial].

## How it works
**Inputs:** OpenGL texture ID, CUDA module containing the surface reference, voxel coordinates, and color/value data [S1, unofficial].
**Outputs:** Updated OpenGL 3D texture contents [S1, unofficial].

**Data Structures:**
*   **Host-side:**
    *   `GLuint glTex`: OpenGL texture object.
    *   `CUgraphicsResource cudaTex`: Registered CUDA graphics resource.
    *   `CUarray cudaArray`: Mapped CUDA array representing the texture memory.
    *   `CUsurfref cudaSurfRef`: Surface reference object linked to the array.
*   **Device-side:**
    *   `surface voxelGrid`: A device-scope surface object bound to the mapped array [S1, unofficial].

**Steps:**
1.  **Create OpenGL Texture:** Generate a texture using `glGenTextures` and bind it as `GL_TEXTURE_3D` [S1, unofficial]. Set minification and magnification filters to `GL_NEAREST` [S1, unofficial]. Allocate storage using `glTexImage3D` with internal format `GL_R8`, format `GL_RED`, and type `GL_FLOAT` [S1, unofficial]. Note: Source [S2, unofficial] describes creating an RGBA8 texture instead.
2.  **Register with CUDA:** Call `cuGraphicsGLRegisterImage` passing the OpenGL texture ID, target `GL_TEXTURE_3D`, and flags `CU_GRAPHICS_REGISTER_FLAGS_SURFACE_LDST` [S1, unofficial]. Source [S2, unofficial] uses `cudaGraphicsRegisterFlagsSurfaceLoadStore` or `cudaGraphicsRegisterFlagsWriteDiscard` depending on usage.
3.  **Map Resource:** Call `cuGraphicsMapResources` (or `cudaGraphicsMapResources`) with the registered resource [S1, unofficial]. Retrieve the mapped CUDA array using `cuGraphicsSubResourceGetMappedArray` (or `cudaGraphicsSubResourceGetMappedArray`) [S1, unofficial]. Unmap immediately if not needed by host code using `cuGraphicsUnmapResources` [S1, unofficial].
4.  **Bind Surface:** Obtain the surface reference from the CUDA module using `cuModuleGetSurfRef` [S1, unofficial]. Set the mapped array to this reference using `cuSurfRefSetArray` [S1, unofficial].
5.  **Launch Kernel:** Launch a CUDA kernel where the device surface `voxelGrid` is active [S1, unofficial]. The kernel writes data directly to the texture coordinates [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Standard OpenGL/CUDA interop API calls are represented below as self-contained functions.
# They perform standard graphics resource allocation, binding, and interop registration.

def allocate_gl_texture():
    """Allocates a new OpenGL texture object ID."""
    return 1

def bind_gl_texture_3d(tex_id):
    """Binds the texture object as a 3D texture target."""
    pass

def set_gl_filter(tex_id, target, filter_mode):
    """Sets the minification/magnification filter for the texture."""
    pass

def allocate_gl_texture_3d(tex_id, internal_format, width, height, depth, format, data_type):
    """Allocates storage for a 3D texture with specified dimensions and formats."""
    pass

def register_gl_texture_with_cuda(tex_id, target, flags):
    """Registers an OpenGL texture with the CUDA graphics interop layer."""
    return 0  # cudaGraphicsResource handle

def map_cuda_resource(resource):
    """Maps the registered CUDA resource for device access."""
    pass

def get_mapped_cuda_array(resource, level, miplevel):
    """Retrieves the CUDA array object mapped to the resource."""
    return 0  # cudaArray handle

def unmap_cuda_resource(resource):
    """Unmaps the CUDA resource, releasing host access."""
    pass

def get_surface_ref_from_module(module, name):
    """Retrieves a surface reference object from a compiled CUDA module."""
    return 0  # CUsurfref handle

def set_surface_array(surf_ref, array, level):
    """Binds a CUDA array to a surface reference."""
    pass

# Host-side setup
glTex = allocate_gl_texture()
bind_gl_texture_3d(glTex)
set_gl_filter(glTex, "GL_TEXTURE_MIN_FILTER", "GL_NEAREST")
set_gl_filter(glTex, "GL_TEXTURE_MAG_FILTER", "GL_NEAREST")
allocate_gl_texture_3d(glTex, "GL_R8", size, size, size, 0, "GL_RED", "GL_FLOAT")

cudaTex = register_gl_texture_with_cuda(glTex, "GL_TEXTURE_3D", "CU_GRAPHICS_REGISTER_FLAGS_SURFACE_LDST")

map_cuda_resource(cudaTex)
cudaArray = get_mapped_cuda_array(cudaTex, 0, 0)
unmap_cuda_resource(cudaTex)

cudaSurfRef = get_surface_ref_from_module(module, "voxelGrid")
set_surface_array(cudaSurfRef, cudaArray, 0)

# Device-side kernel code
# __global__ void kernel() {
#     int3 voxel = ...; // coordinates
#     unsigned char color = ...; // value
#     surf3Dwrite(color, voxelGrid, voxel.x, voxel.y, voxel.z);
# }
```

**Formulas:**
*   Surface Write Operation: $C_{x,y,z} = \text{surf3Dwrite}(color, \text{voxelGrid}, x, y, z)$ [S1, unofficial].

**Parameters:**
*   Texture Format: `GL_R8` / `GL_RED` / `GL_FLOAT` [S1, unofficial].
*   Flags: `CU_GRAPHICS_REGISTER_FLAGS_SURFACE_LDST` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   Precision: The source code example uses `GL_FLOAT` type with `GL_R8` internal format, which is contradictory in standard OpenGL (R8 implies 8-bit integer) [S1, unofficial]. Implementers should verify driver behavior or use consistent formats like `GL_R32F` for float data [S1, unofficial].
*   Resource State: The resource must be unmapped before the OpenGL context uses the texture for rendering, and unmapped before CUDA writes if mapped by another context, though the example unmaps immediately after getting the array [S1, unofficial].

## Subtopics

### CUDA-OpenGL Interop Texture Copy
Copies a 3D volume of float data from device memory into a registered OpenGL 3D texture via CUDA interop, mapping the resource, performing a 3D memory copy, and unmapping [S3, unofficial].
*   **Steps:**
    1.  Map resource: `cudaGraphicsMapResources(1, &m_resource)` [S3, unofficial].
    2.  Get mapped array: `cudaGraphicsSubResourceGetMappedArray(&array, m_resource, 0, 0)` [S3, unofficial].
    3.  Configure copy: `srcPtr` from `make_cudaPitchedPtr(sourcePointer, width*sizeof(float), width, width, height)`, `dstArray` from mapped array, `extent` from `make_extent(width, height, depth)`, `kind` = `cudaMemcpyDeviceToDevice` [S3, unofficial].
    4.  Execute: `cudaMemcpy3D(&copy)` [S3, unofficial].
    5.  Unmap: `cudaGraphicsUnmapResources(1, &m_resource)` [S3, unofficial].
*   **Pitfalls:** Source pointer must not be null and resource must be registered; throws `std::logic_error` if either condition fails [S3, unofficial].

## Operating characteristics
*   **Strengths:** Avoids copying the final image to a temporary device buffer and then to OpenGL, reducing latency and memory bandwidth usage for direct rendering [S2, unofficial].
*   **Limitations:** Requires CUDA and OpenGL interop support. Not given in the sources.
*   **When to use:** Not given in the sources.

## Background
Used in real-time volume rendering engines and scientific visualization tools where high-performance voxelization or ray casting is performed on the GPU and the result is displayed via OpenGL [S1, S2, unofficial].

## Sources
[S1] A Comparison of Optimal Scanline Voxelization Algorithms, https://liu.diva-portal.org/smash/get/diva2:1459330/FULLTEXT01.pdf, fetched 2026-10-25 (unofficial)
[S2] Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, https://github.com/Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, fetched 2026-10-27 (unofficial)
[S3] pvasbob/HZL-fluidEffectsStudio, https://github.com/pvasbob/HZL-fluidEffectsStudio, fetched 2026-10-27 (unofficial)
