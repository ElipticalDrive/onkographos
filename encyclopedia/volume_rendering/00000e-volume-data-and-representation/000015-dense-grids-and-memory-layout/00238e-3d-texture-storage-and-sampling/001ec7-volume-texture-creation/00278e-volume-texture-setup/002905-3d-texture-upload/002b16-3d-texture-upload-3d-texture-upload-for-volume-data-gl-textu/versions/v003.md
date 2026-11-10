# 3D Texture Upload — 3D Texture Upload for Volume Data, GL_TEXTURE_3D volume upload, Volume Texture Binding

Also called: stackImages

## Overview
This method handles the transfer of preprocessed scalar field data from CPU memory to GPU memory via an OpenGL `GL_TEXTURE_3D` object. It solves the problem of bridging the gap between host-side data structures (often nested arrays or raw buffers) and the GPU's texture sampling hardware. It is distinct from [Volume Texture Generation](7099-volume-texture-generation.md), which prepares the data format, and [3D Texture Upload](4419-3d-texture-upload.md), which manages the final RGBA classified data upload with specific filtering parameters.

## How it works
**Inputs and Outputs:**
- Input: A 3D image stack represented as a nested array structure `stack[x][y][z]` containing scalar values, or a sequence of 3D timesteps.
- Output: An OpenGL `GL_TEXTURE_3D` object populated with data, ready for sampling in shaders.

**Data Structures:**
- `stack`: A nested array where `stack[x]` is a 2D plane, `stack[x][y]` is a 1D line, and `stack[x][y][z]` is the scalar value.
- `stackBuffer`: A flat `Uint8Array` (or equivalent byte buffer) of size `sizeX * sizeY * sizeZ`.
- `texID`: The OpenGL texture object ID.

**Numbered Steps:**

1. **Calculate Dimensions:**
   Determine the dimensions of the volume:
   $$sizeX = stack.length$$
   $$sizeY = stack[0].length$$
   $$sizeZ = stack[0][0].length$$

2. **Allocate Buffer:**
   Allocate a flat buffer `stackBuffer` of length `sizeX * sizeY * sizeZ`.

3. **Flatten and Pack Data:**
   Iterate through the nested array and copy values into the flat buffer. The memory order is x-fastest, y-middle, z-slowest.
   $$stride_y = sizeX$$
   $$stride_z = sizeX \times sizeY$$
   
   Pseudocode for flattening:
   ```python
   i = 0
   for z in range(sizeZ):
       for y in range(sizeY):
           for x in range(sizeX):
               stackBuffer[i] = stack[x][y][z]
               i += 1
   ```
   [G] Ensure scalar values are normalized to the `Uint8` range [0, 255] if the source data is floating-point, as `GL_UNSIGNED_BYTE` is used in the upload call.

4. **Create and Bind Texture:**
   Generate an OpenGL texture object and bind it to `GL_TEXTURE_3D`.
   ```python
   texID = glGenTextures(1)
   glBindTexture(GL_TEXTURE_3D, texID)
   ```

5. **Set Texture Parameters:**
   Configure wrapping and filtering. For volume rendering, linear filtering is common for smooth interpolation.
   ```python
   glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE)
   glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE)
   glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_WRAP_R, GL_CLAMP_TO_EDGE)
   glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_MIN_FILTER, GL_LINEAR)
   glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_MAG_FILTER, GL_LINEAR)
   ```

6. **Upload Data:**
   Use `glTexImage3D` to allocate storage and copy the data.
   ```python
   glTexImage3D(
       GL_TEXTURE_3D,      # target
       0,                  # miplevel
       GL_INTENSITY,       # internalformat (or GL_LUMINANCE)
       sizeX,              # width
       sizeY,              # height
       sizeZ,              # depth
       0,                  # border
       GL_LUMINANCE,       # format
       GL_UNSIGNED_BYTE,   # type
       stackBuffer         # data
   )
   ```
   [S1, unofficial] The internal format `GL_INTENSITY` and format `GL_LUMINANCE` are used for single-channel scalar data.

**Edge Cases and Pitfalls:**
- **Precision:** If source data is not `Uint8`, explicit normalization is required before upload to avoid overflow or truncation errors [G].
- **Memory Order:** The flattening loop must strictly follow the x-fastest, y-middle, z-slowest order to match OpenGL's expected layout for `GL_TEXTURE_3D` with these parameters.
- **Boundaries:** `GL_CLAMP_TO_EDGE` wrapping is recommended to prevent sampling artifacts at volume boundaries.

## Subtopics

### Texture Streaming
A variant for handling 4D data (3D volume + time) when graphics memory is insufficient to hold all timesteps. It allocates a single `GL_TEXTURE_3D` and updates it frame-by-frame using `glTexSubImage3D` [S2, unofficial].
- **Steps:**
  1. Allocate one texture ID and bind it.
  2. Upload the first timestep using `glTexImage3D`.
  3. Initialize `curTimeStep = 0`.
  4. In the draw loop, increment `curTimeStep` modulo `numTimesteps`.
  5. Bind the texture and call `glTexSubImage3D` with the current timestep's data.
- **Parameters:** Target `GL_TEXTURE_3D`, Internal Format `GL_INTENSITY` [S2, unofficial].
- **Comparison:** Uses less graphics memory than caching all timesteps but involves CPU `memcpy` and `glTexSubImage3D` calls per frame, causing a download bottleneck [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels, for the flattening and upload phase.
- **Memory Complexity:** $O(N)$ for the host buffer and GPU texture storage.
- **Performance:** Upload speed is limited by PCIe bandwidth. For large volumes, this can be a bottleneck. Texture streaming reduces GPU memory usage but adds per-frame upload overhead [S2, unofficial].
- **Strengths:** Simple implementation; leverages standard OpenGL texture sampling hardware.
- **Limitations:** Requires data conversion on CPU; single-channel upload limits direct RGBA classification support without additional steps [S1, unofficial].

## Background
Originated from standard OpenGL volume rendering techniques. The specific flattening algorithm is documented in community implementations [S1, unofficial]. Texture streaming for 4D data is described in NVIDIA GTC presentations [S2, unofficial].

## Sources
[S1] arisetiawan4601/dicom-volume-renderer, https://github.com/arisetiawan4601/dicom-volume-renderer, fetched 2026-10-27 (unofficial)
[S2] PDF Slide 1, https://www.nvidia.com/content/GTC/documents/1102_GTC09.pdf, fetched 2026-10-29 (unofficial)
