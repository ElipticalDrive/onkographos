# 3D Texture Initialization

Also called: Texture Initialization

## Overview
Allocates GPU memory for a 3D texture, configures sampling parameters, and uploads initial volume data [S1, unofficial]. It differs from [3D Texture Upload](4419-3d-texture-upload.md) by handling texture creation, storage allocation, and mipmap generation, whereas the parent focuses on binding and parameter setting for existing data [S1, unofficial].

## How it works
**Inputs:**
- `tex_size`: Integer dimension of the cubic volume ($N \times N \times N$) [S1, unofficial].
- `data`: Host memory containing volume data, or omitted to use zero-initialized data [S1, unofficial].

**Outputs:**
- `tex3D`: OpenGL texture ID bound to `GL_TEXTURE_3D` [S1, unofficial].

**Data Structures:**
- `data`: A 1D array of floats with size $4 \times tex\_size^3$ [S1, unofficial].

**Steps:**
1. Allocate a float vector `data` of size $4 \times tex\_size^3$ and initialize all elements to $0.0f$ [S1, unofficial].
2. Generate a texture ID `tex3D` and bind it as `GL_TEXTURE_3D` [S1, unofficial].
3. Set texture wrap modes for S, T, and R axes to `GL_CLAMP_TO_BORDER` [S1, unofficial].
4. Set minification and magnification filters to `GL_LINEAR_MIPMAP_LINEAR` [S1, unofficial].
5. Allocate storage for a 10-level mipmap pyramid using `GL_RGBA8` format and dimensions `tex_size` [S1, unofficial].
6. Upload the zero-initialized data to level 0 of the texture [S1, unofficial].
7. Generate mipmaps for the texture [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_3d_texture(tex_size):
    # Step 1: Allocate host data vector of size 4 * tex_size^3 and initialize to 0.0f
    data = [0.0] * (4 * tex_size * tex_size * tex_size)

    # Step 2: Generate texture ID and bind as GL_TEXTURE_3D
    tex3D = generate_texture_id()
    bind_texture(GL_TEXTURE_3D, tex3D)

    # Step 3: Set wrap modes for S, T, R axes to GL_CLAMP_TO_BORDER
    set_texture_parameter(GL_TEXTURE_3D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_BORDER)
    set_texture_parameter(GL_TEXTURE_3D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_BORDER)
    set_texture_parameter(GL_TEXTURE_3D, GL_TEXTURE_WRAP_R, GL_CLAMP_TO_BORDER)

    # Step 4: Set minification and magnification filters to GL_LINEAR_MIPMAP_LINEAR
    set_texture_parameter(GL_TEXTURE_3D, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_LINEAR)
    set_texture_parameter(GL_TEXTURE_3D, GL_TEXTURE_MAG_FILTER, GL_LINEAR_MIPMAP_LINEAR)

    # Step 5: Allocate storage for a 10-level mipmap pyramid using GL_RGBA8 format and dimensions tex_size
    allocate_mipmap_storage(GL_TEXTURE_3D, levels=10, internal_format=GL_RGBA8, width=tex_size, height=tex_size, depth=tex_size)

    # Step 6: Upload zero-initialized data to level 0
    upload_texture_data(GL_TEXTURE_3D, level=0, width=tex_size, height=tex_size, depth=tex_size, format=GL_RGBA, type=GL_FLOAT, data=data)

    # Step 7: Generate mipmaps
    generate_mipmaps(GL_TEXTURE_3D)

    return tex3D

# Helper definitions for OpenGL operations
def generate_texture_id():
    """Returns a unique integer texture ID."""
    pass
def bind_texture(target, texture_id):
    """Binds the texture_id to the target (e.g., GL_TEXTURE_3D)."""
    pass
def set_texture_parameter(target, pname, param):
    """Sets a texture parameter pname to param for the bound target."""
    pass
def allocate_mipmap_storage(target, levels, internal_format, width, height, depth):
    """Allocates storage for a mipmap pyramid with the specified levels, format, and base dimensions."""
    pass
def upload_texture_data(target, level, width, height, depth, format, type, data):
    """Uploads the data array to the specified mipmap level of the bound texture."""
    pass
def generate_mipmaps(target):
    """Generates mipmaps for the bound texture based on the base level data."""
    pass
```

**Parameters:**
- `tex_size`: Typical values depend on resolution requirements. Not given in the sources.
- `mipmap_levels`: 10 levels [S1, unofficial].

**Edge Cases:**
- **Precision Mismatch:** Uploading float data to `GL_RGBA8` storage may result in precision loss or undefined behavior depending on the driver [S1, unofficial].
- **Boundary Conditions:** `GL_CLAMP_TO_BORDER` requires setting the border color using `glTexParameterfv(GL_TEXTURE_3D, GL_TEXTURE_BORDER_COLOR, border_color)` [G]. The source does not specify the border color, so a default of $(0,0,0,0)$ is assumed [G].
- **Degenerate Input:** `tex_size` must be greater than 0 [G]. Mipmap generation requires power-of-two dimensions for some older OpenGL versions, but modern OpenGL supports non-power-of-two [G].

## Subtopics

### CPU Smoke Upload
Calculates a 3D density grid on the CPU via fluid simulation, uploads it to a `Texture3D` resource, and displays it as a smoke-like field using the same volume render path as GPU simulations [S2]. This variant is chosen when simulation data is generated on the CPU rather than the GPU [S2].

## Operating characteristics
- **Time Complexity:** $O(N^3)$ for data initialization and upload [S1, unofficial].
- **Memory Complexity:** $O(N^3)$ for texture storage [S1, unofficial].
- **Performance:** Depends on `tex_size` and transfer bandwidth. Not given in the sources.
- **Strengths:** Simple setup, supports mipmapping for LOD [G].
- **Limitations:** Float-to-integer conversion may cause precision issues [S1, unofficial].
- **When to use:** When initializing a volume texture with known dimensions and data [G].
- **When not to use:** When dynamic updates are frequent and CPU-GPU transfer is a bottleneck [G].

## Background
Originated in OpenGL-based volume rendering pipelines. The specific implementation details are from the `Voxel_Cone_Tracing` repository [S1, unofficial]. The CPU smoke upload variant is from a fluid simulation demo [S2].

## Sources
[S1] kbladin/Voxel_Cone_Tracing, https://github.com/kbladin/Voxel_Cone_Tracing, fetched 2026-10-26 (unofficial)
[S2] Part4 Chapter16 Fluid Simulation Demo, https://github.com/Razria93/Graphics_Study_Archive/issues/33, fetched 2026-10-26
