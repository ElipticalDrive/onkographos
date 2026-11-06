# 3D Texture Upload — 3D Texture Upload for Volume Data, GL_TEXTURE_3D volume upload, Volume Texture Binding

Also called: stackImages

## Overview
This method handles the transfer of classified RGBA volume data from CPU memory to GPU memory by uploading it into an OpenGL `GL_TEXTURE_3D` texture object. It solves the problem of converting nested 3D array structures (common in JavaScript/Python data processing) into a flat, contiguous byte buffer required by OpenGL's `glTexImage3D` or `glTexSubImage3D` functions. It is distinct from [Volume Atlas Texture Update](7157-volume-atlas-texture-update.md), which manages multiple smaller textures in an atlas layout, whereas this method focuses on a single monolithic 3D texture upload.

## How it works
**Inputs:** A 3D image stack represented as a nested array structure `stack[x][y][z]` and the texture dimensions.
**Outputs:** A `Uint8Array` buffer of length $W \times H \times D$ ready for GPU upload, and the configuration of the `GL_TEXTURE_3D` object.

**Data Structures:**
The input is a 3D array where indices correspond to spatial coordinates. The output is a 1D `Uint8Array`.
*   **Memory Layout:** The buffer uses row-major ordering with the X dimension as the fastest-changing index, Y as the middle, and Z as the slowest.
    *   Stride Y ($S_y$) = $W$
    *   Stride Z ($S_z$) = $W \times H$
    *   Offset for element $(x, y, z)$ = $x + y \cdot W + z \cdot W \cdot H$

**Parameters:**
*   $W$: Width (size of X dimension)
*   $H$: Height (size of Y dimension)
*   $D$: Depth (size of Z dimension)
*   Internal format: Typically `GL_RGBA8` or `GL_R8` depending on classification.
*   Format: `GL_RGBA` or `GL_RED`.

**Steps:**
1.  Determine dimensions: $W = \text{stack.length}$, $H = \text{stack}[0].\text{length}$, $D = \text{stack}[0][0].\text{length}$ [S1, unofficial].
2.  Allocate a `Uint8Array` of size $W \times H \times D$ [S1, unofficial].
3.  Iterate through the stack and flatten it into the buffer. The iteration order must match the memory layout (x-fastest) [S1, unofficial].
4.  Bind the texture object and call `glTexImage3D` (or `glTexSubImage3D`) with the flattened buffer [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def upload_volume_texture(stack, gl, texture_id, width, height, depth):
    """
    Uploads a 3D stack to an OpenGL 3D texture.
    stack: list of lists of lists, stack[x][y][z]
    """
    # 1. Allocate flat buffer
    # Size is W * H * D. Each element is 1 byte (Uint8).
    buffer_size = width * height * depth
    stack_buffer = bytearray(buffer_size)
    
    # 2. Flatten the nested array
    # Memory order: x-fastest, y-middle, z-slowest
    # This matches the standard OpenGL 3D texture storage order for GL_RGBA8/GL_R8
    i = 0
    for z in range(depth):
        for y in range(height):
            for x in range(width):
                # Access stack[x][y][z] as per source definition [S1, unofficial]
                # Note: Source [S1, unofficial] defines stack[x][y][z] explicitly.
                # In many languages, arrays are [z][y][x] or [y][x][z].
                # We strictly follow [S1, unofficial]'s indexing: stack[x][y][z].
                value = stack[x][y][z]
                
                # Assuming 8-bit unsigned integer data
                stack_buffer[i] = value & 0xFF
                i += 1
                
    # 3. Upload to GPU
    # Bind texture
    gl.bindTexture(gl.GL_TEXTURE_3D, texture_id)
    
    # Define parameters
    internal_format = gl.GL_RGBA8 # Or GL_R8 depending on data
    format_type = gl.GL_UNSIGNED_BYTE
    
    # Upload
    # glTexImage3D(target, level, internalFormat, width, height, depth, 
    #              border, format, type, pixels)
    gl.texImage3D(
        gl.GL_TEXTURE_3D,
        0,                  # level
        internal_format,    # internalFormat
        width,              # width
        height,             # height
        depth,              # depth
        0,                  # border
        gl.GL_RGBA,         # format (assuming RGBA output)
        gl.GL_UNSIGNED_BYTE,# type
        stack_buffer        # pixels
    )
    
    # Set filtering and wrapping parameters as per parent article [4419-3d-texture-upload.md]
    gl.texParameteri(gl.GL_TEXTURE_3D, gl.GL_TEXTURE_MIN_FILTER, gl.GL_LINEAR)
    gl.texParameteri(gl.GL_TEXTURE_3D, gl.GL_TEXTURE_MAG_FILTER, gl.GL_LINEAR)
    gl.texParameteri(gl.GL_TEXTURE_3D, gl.GL_TEXTURE_WRAP_S, gl.GL_CLAMP_TO_EDGE)
    gl.texParameteri(gl.GL_TEXTURE_3D, gl.GL_TEXTURE_WRAP_T, gl.GL_CLAMP_TO_EDGE)
    gl.texParameteri(gl.GL_TEXTURE_3D, gl.GL_TEXTURE_WRAP_R, gl.GL_CLAMP_TO_EDGE)
```

**Edge Cases and Pitfalls:**
*   **Indexing Mismatch:** Source [S1, unofficial] specifies `stack[x][y][z]`. If the input data is stored as `stack[z][y][x]` (common in NumPy/MATLAB), the flattening loop must be adjusted to transpose the indices, otherwise the volume will appear rotated or sheared.
*   **Data Type:** The method assumes 8-bit unsigned integers (`Uint8`). If the source data is 16-bit or floating point, it must be scaled and cast to 8-bit before upload, or a different `internalFormat` (e.g., `GL_RGBA16F`) and `type` must be used.
*   **Memory Limits:** `GL_TEXTURE_3D` has maximum size limits defined by `GL_MAX_3D_TEXTURE_SIZE`. If $W \times H \times D$ exceeds hardware limits, the upload will fail.

## Subtopics

### Volume Texture Generation
This variant describes the preprocessing pipeline required before upload, specifically for data derived from JSON files [S2, unofficial]. It involves organizing raw data into a 3D array, normalizing values, and converting to RAW format [S2, unofficial].
*   **Definition:** The process of converting preprocessed data into 3D textures for efficient shader access [S2, unofficial].
*   **Steps:**
    1.  Organize data from JSON files into a 3D array with shape $(N_{\text{longitude}}, N_{\text{latitude}}, N_{\text{depth}})$ [S2, unofficial].
    2.  Map data values to the range $[0, 255]$ using linear normalization:
        $$ v' = \frac{(v - v_{\min}) \cdot 255}{v_{\max} - v_{\min}} $$
        where $v$ is the original value, $v'$ is the mapped value, and $v_{\min}/v_{\max}$ are the minimum and maximum data values [S2, unofficial].
    3.  Convert the mapped data into a RAW file format, which is commonly used for volume texture file formats [S2, unofficial].
*   **When to choose:** Use this when the source data is not already in a normalized 8-bit 3D array format, such as when loading scientific data from JSON or CSV files [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(W \times H \times D)$ for the flattening step. The GPU upload time depends on the PCIe bandwidth and the size of the buffer.
*   **Memory Complexity:** $O(W \times H \times D)$ bytes for the CPU-side buffer. The GPU consumes an additional $O(W \times H \times D)$ bytes for the texture storage.
*   **Strengths:** Simple implementation; direct mapping to OpenGL's native 3D texture storage; efficient for direct volume rendering (ray casting) where 3D texture sampling is required.
*   **Limitations:** Requires contiguous memory allocation on the CPU; large volumes may exceed VRAM limits or cause long upload stalls; does not handle compression or sparse data efficiently.
*   **When to use:** For dense volume data that fits in VRAM and requires direct 3D texture sampling in shaders.
*   **When not to use:** For extremely large datasets that exceed VRAM (use texture arrays or streaming), or for sparse data (use sparse voxel octrees).

## Background
The method originates from practical implementations of web-based and desktop volume renderers, such as the `dicom-volume-renderer` repository [S1, unofficial] and WebGPU-based frameworks [S2, unofficial]. It relies on OpenGL's `GL_TEXTURE_3D` primitive, which is a logical extension of 2D textures designed to store three-dimensional spatial data [S2, unofficial]. The specific flattening algorithm (x-fastest, y-middle, z-slowest) is a standard convention in graphics programming to ensure compatibility with OpenGL's expected memory layout for 3D textures.

## Sources
[S1] arisetiawan4601/dicom-volume-renderer, https://github.com/arisetiawan4601/dicom-volume-renderer, fetched 2026-10-27 (unofficial)
[S2] The Implementation of a WebGPU-Based Volume Rendering Framework ... - MDPI, https://www.mdpi.com/2076-3417/15/5/2782, fetched 2026-10-29 (unofficial)
