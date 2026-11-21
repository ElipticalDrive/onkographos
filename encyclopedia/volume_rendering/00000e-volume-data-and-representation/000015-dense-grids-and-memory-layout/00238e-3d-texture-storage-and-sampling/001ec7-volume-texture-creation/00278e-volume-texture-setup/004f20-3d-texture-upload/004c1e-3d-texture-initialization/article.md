# 3D Texture Initialization

Also called: Texture Initialization

## Overview
Allocates GPU memory for a 3D texture, configures sampling parameters (wrapping, filtering, mipmapping), and uploads initial volume data. It sets up the `GL_TEXTURE_3D` object required for subsequent volume rendering passes. It differs from [3D Texture Upload](4419-3d-texture-upload.md) by handling the full lifecycle of texture creation, storage allocation, and mipmap generation, whereas the parent focuses on binding and parameter setting for existing data.

## How it works
**Inputs:**
- `tex_size`: Integer dimension of the cubic volume ($N \times N \times N$).
- `data`: Pointer to host memory containing volume data.

**Outputs:**
- `tex3D`: OpenGL texture ID bound to `GL_TEXTURE_3D`.

**Data Structures:**
- `data`: A 1D array of floats with size $4 \times N^3$. Layout is RGBA, where R, G, B, A correspond to the four channels. Each element is a 32-bit float.

**Steps:**
1. **Allocate Host Data:** Create a float vector `data` of size $4 \times tex\_size^3$. Initialize all elements to $0.0f$ [S1, unofficial].
2. **Generate Texture ID:** Call `glGenTextures(1, &tex3D)` to generate a unique texture identifier [S1, unofficial].
3. **Bind Texture:** Bind `tex3D` to `GL_TEXTURE_3D` using `glBindTexture(GL_TEXTURE_3D, tex3D)` [S1, unofficial].
4. **Set Wrap Modes:** Set the wrap mode for S, T, and R axes to `GL_CLAMP_TO_BORDER` using `glTexParameteri` [S1, unofficial].
5. **Set Filters:** Set minification and magnification filters to `GL_LINEAR_MIPMAP_LINEAR` using `glTexParameteri` [S1, unofficial].
6. **Allocate Storage:** Allocate storage for a 10-level mipmap pyramid. The base level (level 0) has dimensions $tex\_size \times tex\_size \times tex\_size$. The internal format is `GL_RGBA8` [S1, unofficial]. Use `glTexImage3D` for level 0 and `glGenerateMipmap` or manual allocation for subsequent levels. Note: `GL_RGBA8` implies 8-bit integer storage, but input data is float. The source specifies uploading float data to `GL_RGBA8` storage, which may involve implicit conversion or specific driver behavior [S1, unofficial].
7. **Upload Base Level:** Upload the zero-initialized `data` to level 0 of the texture using `glTexSubImage3D` or `glTexImage3D` [S1, unofficial].
8. **Generate Mipmaps:** Generate mipmaps for the texture using `glGenerateMipmap(GL_TEXTURE_3D)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_3d_texture(tex_size, data):
    # Step 1: Allocate and initialize host data
    # data is expected to be a list/array of size 4 * tex_size^3
    # If not provided, initialize to zeros
    if data is None:
        data = [0.0] * (4 * tex_size * tex_size * tex_size)
    
    # Step 2: Generate texture ID
    tex3D = glGenTextures(1)
    
    # Step 3: Bind texture
    glBindTexture(GL_TEXTURE_3D, tex3D)
    
    # Step 4: Set wrap modes
    glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_BORDER)
    glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_BORDER)
    glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_WRAP_R, GL_CLAMP_TO_BORDER)
    
    # Step 5: Set filters
    glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_LINEAR)
    glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_MAG_FILTER, GL_LINEAR_MIPMAP_LINEAR)
    
    # Step 6: Allocate storage for mipmap pyramid
    # Base level: tex_size x tex_size x tex_size, format GL_RGBA8
    # Note: GL_RGBA8 is 8-bit integer, but data is float. 
    # The source [S1, unofficial] specifies this combination.
    glTexImage3D(GL_TEXTURE_3D, 0, GL_RGBA8, tex_size, tex_size, tex_size, 0, GL_RGBA, GL_FLOAT, data)
    
    # Allocate storage for remaining 9 levels of mipmap pyramid
    # This step is implied by "Allocate storage for a 10-level mipmap pyramid"
    # In practice, glGenerateMipmap handles the allocation and generation
    # if the base level is complete.
    
    # Step 7: Upload data (already done in glTexImage3D above for level 0)
    # If data was updated after allocation, use glTexSubImage3D
    
    # Step 8: Generate mipmaps
    glGenerateMipmap(GL_TEXTURE_3D)
    
    return tex3D
```

**Parameters:**
- `tex_size`: Typical values depend on resolution requirements. Not given in the sources.
- `mipmap_levels`: 10 levels [S1, unofficial].

**Edge Cases:**
- **Precision Mismatch:** Uploading float data to `GL_RGBA8` storage may result in precision loss or undefined behavior depending on the driver. The source [S1, unofficial] specifies this, so it is implemented as such.
- **Boundary Conditions:** `GL_CLAMP_TO_BORDER` requires setting the border color using `glTexParameterfv(GL_TEXTURE_3D, GL_TEXTURE_BORDER_COLOR, border_color)` [G]. The source does not specify the border color, so a default of $(0,0,0,0)$ is assumed [G].
- **Degenerate Input:** `tex_size` must be greater than 0. Mipmap generation requires power-of-two dimensions for some older OpenGL versions, but modern OpenGL supports non-power-of-two. The source does not specify this constraint [G].

## Subtopics

### CPU Smoke Upload
Calculates a 3D density grid on the CPU via fluid simulation, uploads it to a `Texture3D` resource, and displays it as a smoke-like field using the same volume render path as GPU simulations [S2]. This variant is chosen when simulation data is generated on the CPU rather than the GPU.

## Operating characteristics
- **Time Complexity:** $O(N^3)$ for data initialization and upload. Mipmap generation is $O(N^3)$ [S1, unofficial].
- **Memory Complexity:** $O(N^3)$ for the texture storage. 10 levels of mipmap pyramid increase memory usage by a factor of approximately $1.33$ compared to a single level [S1, unofficial].
- **Performance:** Depends on `tex_size` and PCIe bandwidth for upload. Not given in the sources.
- **Strengths:** Simple setup, supports mipmapping for LOD.
- **Limitations:** Float-to-integer conversion may cause precision issues. CPU smoke upload requires CPU-GPU synchronization [S2].
- **When to use:** When initializing a volume texture with known dimensions and data.
- **When not to use:** When dynamic updates are frequent and CPU-GPU transfer is a bottleneck.

## Background
Originated in OpenGL-based volume rendering pipelines. The specific implementation details are from the `Voxel_Cone_Tracing` repository [S1, unofficial]. The CPU smoke upload variant is from a fluid simulation demo [S2].

## Sources
[S1] kbladin/Voxel_Cone_Tracing, https://github.com/kbladin/Voxel_Cone_Tracing, fetched 2026-10-26 (unofficial)
[S2] Part4 Chapter16 Fluid Simulation Demo, https://github.com/Razria93/Graphics_Study_Archive/issues/33, fetched 2026-10-26
