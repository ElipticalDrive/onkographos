# 3D Texture Upload — 3D Texture Upload for Volume Data, GL_TEXTURE_3D volume upload, Volume Texture Binding

Also called: stackImages

## Overview
This method converts a 3D image stack from a nested array structure into a flat 1D buffer suitable for GPU upload via OpenGL. It solves the problem of mapping discrete volumetric data, which is often stored in host-memory structures with varying strides, into a contiguous memory layout required by `glTexImage3D` or `glTexSubImage3D`. Unlike sparse storage methods that suffer from pointer overhead or out-of-core streaming that incurs decoding latency, this method packs data into dense 3D texture slices, trading memory efficiency for hardware compatibility and sampling speed [S1, S2, unofficial]. It differs from its parent [3D Texture Upload](4419-3d-texture-upload.md) by handling the data transformation (format conversion, normalization) prior to the final buffer upload step, and from [3D Texture Initialization](13158-3d-texture-initialization.md) by focusing on the data layout rather than texture object creation.

## How it works
**Inputs and Outputs:**
- Input: A 3D image stack represented as a nested array `stack`, where `stack[x][y][z]` accesses the scalar value at coordinates $(x, y, z)$.
- Output: A flat `Uint8Array` (or equivalent 1D byte buffer) of length $sizeX \times sizeY \times sizeZ$.

**Data Structures and Layout:**
The output buffer is a contiguous block of memory. The memory order is x-fastest, y-middle, z-slowest.
- Stride in Y: $stride_y = sizeX$
- Stride in Z: $stride_z = sizeX \times sizeY$
- Index $i$ for coordinate $(x, y, z)$ is calculated as:
  $$ i = x + y \cdot sizeX + z \cdot sizeX \cdot sizeY $$

**Parameters:**
- `sizeX`: Width of the volume ($stack.length$).
- `sizeY`: Height of the volume ($stack[0].length$).
- `sizeZ`: Depth of the volume ($stack[0][0].length$).

**Steps:**
1. Initialize size variables: $sizeX = stack.length$, $sizeY = stack[0].length$, $sizeZ = stack[0][0].length$.
2. Allocate a buffer `stackBuffer` of size $sizeX \times sizeY \times sizeZ$.
3. Iterate through the volume dimensions in the order $z$, $y$, $x$.
4. For each coordinate $(x, y, z)$, assign `stack[x][y][z]` to `stackBuffer[i]` and increment $i$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def upload_volume_stack(stack):
    # 1. Initialize size variables
    size_x = len(stack)
    size_y = len(stack[0])
    size_z = len(stack[0][0])
    
    # 2. Allocate stackBuffer of size sizeX * sizeY * sizeZ
    # Assuming Uint8Array or equivalent byte buffer
    stack_buffer = bytearray(size_x * size_y * size_z)
    
    # 3. Iterate z from 0 to sizeZ, y from 0 to sizeY, x from 0 to sizeX
    i = 0
    for z in range(size_z):
        for y in range(size_y):
            for x in range(size_x):
                # 4. Assign stack[x][y][z] to stackBuffer[i]
                # Note: Input stack is assumed to be pre-converted to uint8 values
                # if not, explicit normalization/clamping is required here [G]
                stack_buffer[i] = stack[x][y][z]
                i += 1
                
    return stack_buffer
```

**Edge Cases and Pitfalls:**
- **Precision:** The source assumes `Uint8Array` [S1, unofficial]. If the source data is floating-point, it must be normalized to $[0, 255]$ and cast to unsigned byte before upload. Failure to do so results in undefined behavior or incorrect rendering.
- **Boundaries:** The code assumes a rectangular grid. Non-rectangular volumes require masking or padding.
- **Degenerate Input:** If any dimension is 0, the buffer size is 0, and no upload occurs.

## Subtopics

### [Volume Texture Generation](7099-volume-texture-generation.md)
Volume Texture Generation converts raw volumetric data arrays into GPU-accessible 3D textures (volume textures) suitable for direct volume rendering. It solves the problem of mapping discrete scalar field data into a format that graphics hardware can sample efficiently via shaders. Unlike sparse storage methods that suffer from pointer overhead or out-of-core streaming that incurs decoding latency, this method packs data into dense 3D texture slices, trading memory efficiency for hardware compatibility and sampling speed. It differs from its parent [3D Texture Upload](4890-3d-texture-upload-3d-texture-upload-for-volume-data-gl-textu.md) by handling the data transformation (format conversion, normalization) prior to the final buffer upload step.

### [VTC 3D Texture De-interleaving](13884-vtc-3d-texture-de-interleaving.md)
VTC 3D Texture De-interleaving converts NVIDIA's Volume Texture Compression (VTC) block-compressed 3D texture data from its native interleaved memory layout into a contiguous, slice-by-slice layout suitable for standard GPU volume upload. It solves the problem where DXT-compressed volumes were incorrectly treated as 2D textures (using only the first slice-sized bytes), causing sampling artifacts such as dark translucent squares with hard edges due to the interleaved block structure. It sets itself apart from generic volume uploads by specifically handling the spatial interleaving of compressed blocks across depth layers defined in the `NV_texture_compression_vtc` specification.

### Texture Streaming
**Step:**
1. Allocate one 3D texture ID.
2. Bind the texture, set parameters, and upload the first timestep's data using `glTexImage3D`.
3. Initialize `curTimeStep` to 0.
4. During draw, increment `curTimeStep` modulo `numTimesteps`, bind the texture, and update it with the current timestep's data using `glTexSubImage3D`.

**Parameter:**
- Texture target: `GL_TEXTURE_3D`.
- Internal format: `GL_INTENSITY`.

**Comparison:**
Compared to caching, this method uses less graphics memory but involves CPU memcpy and `glTexSubImage3D` calls for each frame, causing a download bottleneck [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(N)$ where $N$ is the total number of voxels ($sizeX \times sizeY \times sizeZ$) for the flattening step. The subsequent OpenGL upload is $O(N)$ but bounded by PCIe bandwidth.
- Memory: $O(N)$ for the host-side buffer.

**Measured Performance:**
Not given in the sources.

**Scaling:**
Performance scales linearly with volume size. For large volumes, the bottleneck is typically the PCIe bus transfer rather than the CPU flattening.

**Strengths and Limitations:**
- Strengths: Simple implementation, compatible with standard OpenGL pipelines, efficient for static volumes.
- Limitations: Requires full volume upload for static data; for dynamic data, naive streaming causes CPU/GPU synchronization stalls.

**When to use it and when not:**
Use for static volumes or when the volume fits entirely in GPU memory. Do not use for very large dynamic volumes without optimization (e.g., PBOs) due to the download bottleneck described in Texture Streaming [S2, unofficial].

## Background
This method is a fundamental step in OpenGL-based volume rendering pipelines, enabling the transfer of host-side volumetric data to the GPU. It is commonly found in DICOM volume renderers and educational graphics resources [S1, S2, unofficial].

## Sources
[S1] arisetiawan4601/dicom-volume-renderer, https://github.com/arisetiawan4601/dicom-volume-renderer, fetched 2026-10-27 (unofficial)
[S2] PDF Slide 1, https://www.nvidia.com/content/GTC/documents/1102_GTC09.pdf, fetched 2026-10-29 (unofficial)
