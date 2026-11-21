# Texture Streaming

Also called: GPUScaledTexture management, Vispy volume texture management

## Overview
Texture streaming is a technique for rendering 4D volumetric data (3D spatial dimensions + time) when the total volume exceeds available graphics memory. It solves the memory constraint by allocating a single 3D texture and sequentially uploading individual timesteps from main memory to the GPU for each frame. Unlike caching all timesteps in VRAM, this method trades increased CPU-GPU bus traffic for reduced graphics memory usage.

## How it works
**Inputs and Outputs:**
- Input: A 4D array of volume data `pData` with dimensions `[numTimesteps][dimx][dimy][dimz]`, stored in main memory.
- Output: A rendered frame using the current timestep's volume data sampled from a bound 3D texture.

**Data Structures:**
- `texID`: A single OpenGL texture ID (`GLuint`).
- `curTimeStep`: An integer index tracking the current frame's timestep, initialized to 0.
- `numTimesteps`: The total number of timesteps in the dataset.
- `dimx, dimy, dimz`: The spatial dimensions of the volume.

**Parameters:**
- Texture target: `GL_TEXTURE_3D` [S2, unofficial].
- Internal format: `GL_INTENSITY` [S2, unofficial].
- Pixel format: `GL_LUMINANCE` [S2, unofficial].
- Pixel type: `GL_UNSIGNED_BYTE` [S2, unofficial].
- Texture parameters (wrap, filter): Set via `glTexParameteri` [S2, unofficial].

**Steps:**
1. **Initialization:**
   - Allocate one 3D texture ID using `glGenTextures`.
   - Bind the texture to `GL_TEXTURE_3D`.
   - Set texture parameters (wrap modes, filtering) using `glTexParameteri`.
   - Upload the first timestep (`pData[0]`) to the GPU using `glTexImage3D`. This allocates the texture storage.
   - Initialize `curTimeStep = 0`.

2. **Rendering Loop (per frame):**
   - Increment `curTimeStep` and wrap it around: `curTimeStep = (curTimeStep + 1) % numTimesteps`.
   - Bind the texture to `GL_TEXTURE_3D`.
   - Upload the current timestep's data (`pData[curTimeStep]`) to the existing texture storage using `glTexSubImage3D`. This overwrites the previous timestep's data in place.
   - Execute the drawing code (ray marching or similar) which samples from the bound texture.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
texID = glGenTextures(1)
glBindTexture(GL_TEXTURE_3D, texID)
glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE)
glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE)
glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_WRAP_R, GL_CLAMP_TO_EDGE)
glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_MIN_FILTER, GL_LINEAR)
glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_MAG_FILTER, GL_LINEAR)

# Upload first timestep
glTexImage3D(
    GL_TEXTURE_3D, 0, GL_INTENSITY,
    dimx, dimy, dimz, 0,
    GL_LUMINANCE, GL_UNSIGNED_BYTE,
    pData[0]
)
curTimeStep = 0

# Per-frame rendering
def render_frame(pData, numTimesteps, dimx, dimy, dimz):
    global curTimeStep
    curTimeStep = (curTimeStep + 1) % numTimesteps
    
    glBindTexture(GL_TEXTURE_3D, texID)
    
    # Upload current timestep data
    glTexSubImage3D(
        GL_TEXTURE_3D, 0, 0, 0, 0,
        dimx, dimy, dimz,
        GL_LUMINANCE, GL_UNSIGNED_BYTE,
        pData[curTimeStep]
    )
    
    # Call drawing code here
    draw_volume()
```

**Edge Cases and Pitfalls:**
- **Precision:** The example uses `GL_UNSIGNED_BYTE` and `GL_INTENSITY`, limiting data to 8-bit single-channel. For higher precision or multi-channel data, formats must be adjusted, though multi-channel support requires shader modifications to expose multiple channels [S1, unofficial].
- **Performance:** The method involves a CPU `memcpy` and a `glTexSubImage3D` call per frame, creating a potential download bottleneck [S2, unofficial].
- **Multi-channel Data:** Precomputing results for custom functions across channel dimensions (e.g., for brainbow images with high color variability) is slow and inconvenient, requiring near-continuous precomputation [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per frame for texture upload, but dependent on the bandwidth of the CPU-GPU bus. The cost is proportional to the size of one timestep's volume data.
- **Memory Complexity:** $O(V)$ where $V$ is the size of a single 3D volume, regardless of the number of timesteps. This is significantly lower than caching all timesteps ($O(T \times V)$).
- **Measured Performance:** Involves CPU `memcpy` and `glTexSubImage3D` calls for each frame, causing a download bottleneck [S2, unofficial].
- **Strengths:** Minimal graphics memory usage; suitable for large 4D datasets that do not fit in VRAM.
- **Limitations:** High CPU-GPU bus traffic; potential frame rate reduction due to data transfer latency; not suitable for real-time interaction if the transfer time exceeds the frame budget.
- **When to use:** When graphics memory is insufficient to hold all timesteps simultaneously, but the dataset fits in main RAM.
- **When not to use:** When low latency and high frame rates are critical, or when the dataset fits entirely in VRAM (caching is preferred).

## Background
The technique is described in NVIDIA GTC 2009 slides as a method for handling 4D data when graphics memory is limited [S2, unofficial]. It is also referenced in the context of Vispy volume visual management, where `GPUScaledTexture` is used to manage single-channel 3D textures for volume rendering [S1, unofficial].

## Sources
[S1] MultiChannels Volume Rendering support, https://github.com/napari/napari/issues/3973, fetched 2026-10-27 (unofficial)
[S2] PDF Slide 1, https://www.nvidia.com/content/GTC/documents/1102_GTC09.pdf, fetched 2026-10-29 (unofficial)
