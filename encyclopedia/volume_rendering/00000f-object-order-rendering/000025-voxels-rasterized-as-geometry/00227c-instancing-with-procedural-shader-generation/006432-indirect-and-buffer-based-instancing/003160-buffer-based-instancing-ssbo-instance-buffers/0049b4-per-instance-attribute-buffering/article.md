# Per-Instance Attribute Buffering

Also called: Regular Vertex Buffers, Textures and Texture Buffers, Uniform Buffers

## Overview
Per-Instance Attribute Buffering supplies per-instance data to vertex shaders using standard graphics buffer types instead of dedicated instancing mechanisms [S1, unofficial]. It provides three distinct approaches—uniform buffers, texture buffers, and regular vertex buffers—each handling data distribution and size constraints differently [S1, unofficial].

## How it works
### Inputs and Outputs
*   **Input:** Per-instance data (e.g., transformation matrices, visibility flags, texture indices) [S1, unofficial].
*   **Output:** Transformed vertex positions and attributes for rasterization [S1, unofficial].

### Data Structures and Layouts
1.  **Uniform Buffers:** Expose per-instance data as regular shader uniform array variables [S1, unofficial]. The maximum size is typically limited to 64 KB [S1, unofficial].
2.  **Textures and Texture Buffers:** Embed per-instance data into texels, retrieved via texel-fetching instructions [S1, unofficial]. Data must be converted and encoded according to the pixel description of the texture [S1, unofficial]. The maximum size is constrained by video memory and the maximum texture resolution and texel format supported by the hardware [S1, unofficial].
3.  **Regular Vertex Buffers:** Feed per-instance data as regular vertex attribute shader variables [S1, unofficial]. The hardware loads per-instance data into registers prior to invoking the vertex program, dispensing the need for an instance identifier [S1, unofficial]. The size is restricted only by available video memory [S1, unofficial].

### Numbered Steps
1.  **Data Preparation:** Encode per-instance data into the chosen buffer format [S1, unofficial].
2.  **Buffer Binding:** Bind the buffer to the appropriate shader stage (uniform block, texture unit, or vertex attribute array) [S1, unofficial].
3.  **Draw Call Issuance:** For uniform buffers exceeding the size limit, issue multiple draw calls on multiple buffers or subregions [S1, unofficial]. For texture and vertex buffers, issue a single draw call [G].
4.  **Shader Execution:** The vertex shader accesses data via uniform arrays, texel fetches, or hardware-loaded registers [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_per_instance_attribute_buffering(instance_data, base_geometry, method):
    # 1. Prepare data in the chosen buffer format
    if method == "uniform":
        # Uniform buffers are limited to ~64 KB; split data if larger
        buffers = []
        chunk_size = 64 * 1024
        for i in range(0, len(instance_data), chunk_size):
            buffers.append(instance_data[i:i+chunk_size])
    elif method == "texture":
        # Texture buffers require encoding per pixel format
        # Not given in the sources.
        buffers = [instance_data]
    elif method == "vertex_buffer":
        # Vertex buffers scale with available video memory
        buffers = [instance_data]
    else:
        raise ValueError("Unsupported method")

    # 2. Issue draw calls and execute shader
    for buf in buffers:
        # Hardware automatically loads vertex buffer attributes into registers
        # For textures/uniforms, the shader fetches data using the instance index
        # Not given in the sources.
        pass

def vertex_shader_access(instance_index, method):
    # Access per-instance data based on the chosen method
    if method == "uniform":
        data = uniform_array[instance_index]
    elif method == "texture":
        data = texture_buffer[instance_index]
    elif method == "vertex_buffer":
        data = vertex_attribute_array[instance_index]
    return data
```

### Edge Cases and Pitfalls
*   **Uniform Buffer Size:** Exceeding the ~64 KB limit requires multiple draw calls or buffer subregions [S1, unofficial].
*   **Texture Encoding:** Data must be correctly encoded according to the pixel format, adding conversion overhead [S1, unofficial].
*   **Hardware Loading:** Regular vertex buffers bypass explicit instance ID lookups, but require the hardware to manage register allocation [S1, unofficial].

### Subtopics
#### Uniform Buffers
Exposes per-instance data as regular shader uniform array variables [S1, unofficial]. Limited to approximately 64 KB, requiring multiple draw calls for larger datasets [S1, unofficial].

#### Textures and Texture Buffers
Embeds per-instance data into texels retrieved via texel-fetching instructions [S1, unofficial]. Requires data conversion and encoding according to the pixel description [S1, unofficial].

#### Regular Vertex Buffers
Feeds per-instance data as regular vertex attribute shader variables [S1, unofficial]. The hardware loads data into registers prior to invoking the vertex program, dispensing the need for an instance identifier [S1, unofficial]. Size is restricted only by available video memory [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [S1, unofficial]
*   **Memory Complexity:** Not given in the sources. [S1, unofficial]
*   **Performance:** Regular vertex buffers avoid dynamic instance ID lookups [S1, unofficial]. Uniform buffers are fast but size-limited [S1, unofficial]. Texture buffers require encoding overhead [S1, unofficial].
*   **Strengths:** No need for explicit instance ID management in the shader for vertex buffers; hardware handles data loading [S1, unofficial].
*   **Limitations:** Uniform buffers have strict size limits; texture buffers require data encoding [S1, unofficial].

## Background
This method is described in the context of hardware-accelerated geometry instancing for surfel rendering [S1, unofficial]. It represents an alternative to dedicated instancing mechanisms by utilizing existing buffer types [S1, unofficial].

## Sources
[S1] Hardware-accelerated geometry instancing for surfel and voxel rendering of scanned 4d media, http://133.5.19.150/papers/2010_2013/1-1-36.pdf, fetched 2026-10-31 (unofficial)
