# 3D Texture Upload — 3D Texture Upload for Volume Data, GL_TEXTURE_3D volume upload, Volume Texture Binding

Also called: stackImages

## Overview
This method converts a 3D image stack from a nested array structure into a flat 1D `Uint8Array` buffer for GPU upload into an OpenGL `GL_TEXTURE_3D` texture [S1, unofficial]. It is distinct from [Volume Atlas Texture Update](7157-volume-atlas-texture-update.md), which manages multiple smaller textures in an atlas layout, whereas this method focuses on a single monolithic 3D texture upload [G].

## How it works
**Inputs:** A 3D image stack represented as a nested array structure and the texture dimensions [S1, unofficial].
**Outputs:** A `Uint8Array` buffer of length $W \times H \times D$ ready for GPU upload [S1, unofficial].

**Data Structures:**
*   **Memory Layout:** The buffer uses x-fastest, y-middle, z-slowest ordering [S1, unofficial].
    *   Stride Y ($S_y$) = $W$
    *   Stride Z ($S_z$) = $W \times H$
    *   Offset for element $(x, y, z)$ = $x + y \cdot W + z \cdot W \cdot H$ [S1, unofficial].
*   **Data Type:** The method assumes 8-bit unsigned integers (`Uint8`) [S1, unofficial].

**Parameters:**
*   $W$: Width (size of X dimension)
*   $H$: Height (size of Y dimension)
*   $D$: Depth (size of Z dimension)

**Steps:**
1.  Determine dimensions: $W = \text{stack.length}$, $H = \text{stack}[0].\text{length}$, $D = \text{stack}[0][0].\text{length}$ [S1, unofficial].
2.  Allocate a `Uint8Array` of size $W \times H \times D$ [S1, unofficial].
3.  Iterate through the stack and flatten it into the buffer, respecting the x-fastest memory layout [S1, unofficial].
4.  Bind the texture object and call `glTexImage3D` (or `glTexSubImage3D`) with the flattened buffer [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def flatten_volume_stack(stack, width, height, depth):
    """
    Converts a nested 3D stack into a flat 1D Uint8Array buffer.
    stack: list of lists of lists, indexed as stack[x][y][z] per source notes.
    """
    size_x = width
    size_y = height
    size_z = depth
    
    # Allocate stackBuffer of size sizeX * sizeY * sizeZ
    stack_buffer = bytearray(size_x * size_y * size_z)
    
    # Iterate z from 0 to sizeZ, y from 0 to sizeY, x from 0 to sizeX
    i = 0
    for z in range(size_z):
        for y in range(size_y):
            for x in range(size_x):
                # Assign stack[x][y][z] to stackBuffer[i] and increment i
                stack_buffer[i] = stack[x][y][z]
                i += 1
                
    return stack_buffer
```

**Edge Cases and Pitfalls:**
*   **Data Type:** The method assumes 8-bit unsigned integers (`Uint8`). If the source data is 16-bit or floating point, it must be scaled and cast to 8-bit before upload, or a different `internalFormat` (e.g., `GL_RGBA16F`) and `type` must be used [G].
*   **Memory Limits:** `GL_TEXTURE_3D` has maximum size limits defined by `GL_MAX_3D_TEXTURE_SIZE`. If $W \times H \times D$ exceeds hardware limits, the upload will fail [G].

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

### Texture Streaming
This variant manages time-varying volume data by cycling through timesteps using a single 3D texture [S3, unofficial].
*   **Steps:**
    1.  Allocate one 3D texture ID and bind it [S3, unofficial].
    2.  Set texture parameters (wrap, filter) and upload the first timestep's data using `glTexImage3D` with target `GL_TEXTURE_3D` and internal format `GL_INTENSITY` [S3, unofficial].
    3.  Initialize `curTimeStep` to 0 [S3, unofficial].
    4.  During each draw call, increment `curTimeStep` modulo `numTimesteps`, bind the texture, and update it with the current timestep's data using `glTexSubImage3D` [S3, unofficial].
*   **Comparison:** Compared to caching all timesteps, this method uses less graphics memory but involves CPU `memcpy` and `glTexSubImage3D` calls for each frame, causing a download bottleneck [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(W \times H \times D)$ for the flattening step [G]. The GPU upload time depends on the PCIe bandwidth and the size of the buffer [G].
*   **Memory Complexity:** $O(W \times H \times D)$ bytes for the CPU-side buffer [G]. The GPU consumes an additional $O(W \times H \times D)$ bytes for the texture storage [G].
*   **Strengths:** Simple implementation; direct mapping to OpenGL's native 3D texture storage; efficient for direct volume rendering (ray casting) where 3D texture sampling is required [G].
*   **Limitations:** Requires contiguous memory allocation on the CPU; large volumes may exceed VRAM limits or cause long upload stalls; does not handle compression or sparse data efficiently [G].
*   **When to use:** For dense volume data that fits in VRAM and requires direct 3D texture sampling in shaders [G].
*   **When not to use:** For extremely large datasets that exceed VRAM (use texture arrays or streaming), or for sparse data (use sparse voxel octrees) [G].

## Background
The method relies on OpenGL's `GL_TEXTURE_3D` primitive, which is a logical extension of traditional 2D textures designed to store three-dimensional spatial data [S2, unofficial]. It is commonly used in web-based and desktop volume renderers to transfer classified volume data from CPU memory to GPU memory [G].

## Sources
[S1] arisetiawan4601/dicom-volume-renderer, https://github.com/arisetiawan4601/dicom-volume-renderer, fetched 2026-10-27 (unofficial)
[S2] The Implementation of a WebGPU-Based Volume Rendering Framework ... - MDPI, https://www.mdpi.com/2076-3417/15/5/2782, fetched 2026-10-29 (unofficial)
[S3] PDF Slide 1, https://www.nvidia.com/content/GTC/documents/1102_GTC09.pdf, fetched 2026-10-29 (unofficial)
