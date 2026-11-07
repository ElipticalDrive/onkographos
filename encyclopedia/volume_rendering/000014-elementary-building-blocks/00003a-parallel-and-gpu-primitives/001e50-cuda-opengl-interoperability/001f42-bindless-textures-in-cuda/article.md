# Bindless Textures in CUDA

Also called: Bindless Textures, BindlessTexture objects

## Overview
Bindless Textures in CUDA is a method for managing and accessing texture data directly from CUDA kernels without the overhead of OpenGL-CUDA interoperability binding and mapping cycles [S1, S2]. It solves the performance bottlenecks and resource limitations inherent in traditional OpenGL-CUDA texture sharing, where textures must be bound, mapped to CUDA resources, and unmapped every frame [S1, S2]. Unlike [CUDA-OpenGL Interop Surface Write](2228-cuda-opengl-interop-surface-write.md) which focuses on surface buffers, this method specifically addresses texture sampling, allowing a dynamic number of textures to be exposed simultaneously to the kernel at runtime [S1, S2].

## How it works
**Inputs:** Raw texture data (e.g., image files, volumetric slices) and format specifications (channel type, internal format).
**Outputs:** A collection of `BindlessTexture` objects stored in device-accessible memory, ready for sampling in CUDA kernels.

**Data Structures:**
1.  **BindlessTexture Object:** A C++ templated class that wraps the CUDA texture object descriptor. It supports various channel types (e.g., `unsigned char`, `unsigned short`) and internal formats (e.g., `float1`, `float4`, `uchar4`) [S1, S2].
2.  **Texture Registry:** A linked list storing all loaded `BindlessTexture` objects and their associated mipmap arrays to facilitate cleanup [S1, S2].

**Steps:**
1.  **Create Texture Objects:** Use the CUDA API to create mipmapped texture objects in VRAM [S1, S2].
2.  **Asynchronous Mipmap Generation:** Launch a simple CUDA kernel asynchronously to compute mipmaps for each texture during the loading phase [S1, S2].
3.  **Wrap in BindlessTexture:** Instantiate `BindlessTexture` objects using C++ templates to match the specific data format and internal structure of the loaded texture [S1, S2].
4.  **Register for Management:** Add the newly created `BindlessTexture` objects to a linked list to track them for memory management [S1, S2].
5.  **Sample in Kernel:** Access the texture in device code using the `tex` variable descriptor associated with the `BindlessTexture` object [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for loading and managing bindless textures

class BindlessTexture:
    """
    C++ templated wrapper for CUDA texture objects.
    Supports formats like unsigned char, unsigned short, float1, float4, uchar4.
    """
    def __init__(self, cuda_texture_object, format_type, internal_format):
        self.cuda_texture_object = cuda_texture_object
        self.format_type = format_type
        self.internal_format = internal_format
        self.mipmaps = [] # Array of mipmap levels

    def sample(self, coords, lod=0):
        """
        Sample the texture at given coordinates.
        In actual CUDA C++, this would use tex2D or tex3D with the bound texture object.
        Here, we assume a generic sampling function provided by the CUDA runtime.
        """
        # [G] Actual implementation relies on CUDA texture reference binding.
        # The 'tex' variable in device code refers to this object's descriptor.
        return cuda_sample_texture(self.cuda_texture_object, coords, lod)

    def free(self):
        """Free the texture object and its mipmaps."""
        for mipmap in self.mipmaps:
            cuda_free(mipmap)
        cuda_destroy_texture_object(self.cuda_texture_object)

# Global registry for cleanup
texture_registry = []

def load_bindless_texture(raw_data, width, height, format_type, internal_format):
    """
    Load a texture into VRAM and create a BindlessTexture object.
    """
    # 1. Create mipmapped texture object using CUDA API
    # [G] cudaCreateTextureObject is the underlying CUDA API call.
    texture_object = cuda_create_texture_object(raw_data, width, height, format_type, internal_format)
    
    # 2. Asynchronously compute mipmaps
    # [G] A simple kernel is launched to generate mipmap levels.
    # The kernel operates on the texture object's base level.
    cuda_launch_mipmap_kernel(texture_object)
    cuda_device_synchronize() # Ensure mipmaps are ready before exposing
    
    # 3. Create BindlessTexture wrapper
    bindless_tex = BindlessTexture(texture_object, format_type, internal_format)
    
    # 4. Store in linked list for tracking
    texture_registry.append(bindless_tex)
    
    return bindless_tex

def cleanup_bindless_textures():
    """Free all textures in the registry."""
    for tex in texture_registry:
        tex.free()
    texture_registry.clear()

# Usage in CUDA Kernel
# __global__ void render_kernel(BindlessTexture* textures, int num_textures, ...) {
#     int id = blockIdx.x * blockDim.x + threadIdx.x;
#     if (id < num_textures) {
#         // Access texture using the 'tex' variable descriptor
#         // The specific 'tex' variable is bound to the texture object within BindlessTexture
#         float4 color = textures[id].sample(float2(x, y));
#     }
# }
```

**Parameters:**
*   `tex`: The variable name used to read a texture from device code. Only this value needs to be known by the kernel [S1, S2].

**Edge Cases and Pitfalls:**
*   **Overhead:** Traditional OpenGL-CUDA texture mapping requires binding every frame, mapping to a CUDA resource prior to kernel launch, and unbinding/unmapping after, which adds significant overhead [S2].
*   **Resource Limits:** The traditional method limits the number of textures that can be simultaneously loaded [S2].
*   **Hardcoding:** The traditional method requires all CUDA resources to be declared in global scope, meaning the number of textures used needs to be hardcoded [S1, S2].
*   **Cleanup:** Textures must be explicitly freed on program closure or when no longer needed to prevent memory leaks [S1, S2].

## Operating characteristics
**Time and Memory Complexity:**
*   **Loading:** Linear with respect to the number of textures and their resolution, plus the cost of asynchronous mipmap generation.
*   **Sampling:** Constant time per sample, independent of the total number of textures loaded, as each texture is accessed via its unique descriptor.
*   **Memory:** VRAM usage is proportional to the total size of all loaded textures and their mipmaps. The linked list overhead is negligible.

**Measured Performance:**
*   Eliminates the per-frame binding and mapping overhead associated with OpenGL-CUDA interoperability [S1, S2].
*   Allows for a dynamic number of textures to be specified at runtime [S1, S2].
*   Allows all textures to be exposed to the kernel simultaneously [S1, S2].

**Strengths and Limitations:**
*   **Strengths:** High performance due to elimination of binding overhead; flexibility in the number of textures; no need to hardcode texture counts in global scope.
*   **Limitations:** Requires careful memory management (linked list cleanup); relies on CUDA API for texture creation, which may have a steeper learning curve than OpenGL texture management.

**When to Use:**
*   Use when rendering scenes with a dynamic or large number of textures that need to be accessed from CUDA kernels.
*   Use when performance is critical and the overhead of OpenGL-CUDA binding is unacceptable.

**When Not to Use:**
*   Use [CUDA-OpenGL Interop Surface Write](2228-cuda-opengl-interop-surface-write.md) if the primary goal is to write surface buffers directly to OpenGL for immediate display without complex texture sampling.
*   Use traditional OpenGL-CUDA interoperability if the number of textures is small, fixed, and known at compile time, and if the simplicity of OpenGL texture management is preferred over the performance gains.

## Background
Bindless Textures in CUDA were introduced as a more performant design to replace the drawbacks of using OpenGL to manage textures for CUDA kernels [S1, S2]. The traditional method, similar to typical raster-based renderers, was found unsuitable for advanced raymarching renderers due to binding overhead, limited simultaneous texture counts, and the need to hardcode texture numbers in global scope [S1, S2]. The bindless approach utilizes an object-oriented design with C++ templating to support various formats and internal formats, enabling dynamic runtime specification and simultaneous exposure of all textures to the kernel [S1, S2].

## Sources
[S1] Raymarching Distance Fields with CUDA, https://doi.org/10.3390/electronics10222730, fetched 2026-10-26
[S2] Raymarching Distance Fields with CUDA, https://www.mdpi.com/2079-9292/10/22/2730, fetched 2026-10-27
