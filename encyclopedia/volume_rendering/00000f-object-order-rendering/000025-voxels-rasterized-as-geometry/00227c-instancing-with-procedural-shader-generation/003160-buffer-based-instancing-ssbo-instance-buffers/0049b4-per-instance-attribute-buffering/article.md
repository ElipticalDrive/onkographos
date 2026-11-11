# Per-Instance Attribute Buffering

Also called: Regular Vertex Buffers, Textures and Texture Buffers, Uniform Buffers

## Overview
Per-Instance Attribute Buffering is a method for supplying per-instance data to vertex shaders by leveraging standard graphics API mechanisms other than the dedicated hardware instancing SSBO lookup. It addresses the problem of distributing instance-specific parameters (position, scale, rotation) to a single draw call without relying on `gl_InstanceID`-based indexing into a large structured buffer [S1, unofficial]. It differs from its parent [Buffer-Based Instancing (SSBO/Instance Buffers)](6501-buffer-based-instancing-ssbo-instance-buffers.md) by offloading the data loading and indexing logic to the hardware or driver, rather than performing dynamic memory lookups in the vertex shader [S1, unofficial].

## How it works
The method consists of three distinct sub-techniques, each utilizing a different buffer type to store per-instance attributes. The vertex shader receives this data either as uniform arrays or vertex attributes, eliminating the need for explicit instance ID lookups.

### Inputs and Outputs
*   **Input:** Per-instance data (e.g., transformation matrices, visibility flags, texture indices).
*   **Output:** Transformed vertex positions and attributes for rasterization.

### Data Structures and Layouts

#### 1. Uniform Buffers
*   **Layout:** Shader uniform arrays (e.g., `uniform vec3 instancePos[MAX_INSTANCES]`).
*   **Size Limit:** Typically limited to 64 KB [S1, unofficial].
*   **Access:** Data is exposed as regular shader uniform variables. The application must manage buffer splitting if data exceeds the limit.

#### 2. Textures and Texture Buffers
*   **Layout:** Per-instance data encoded into texels. The data must be converted and packed according to the texture's pixel format (e.g., `RGBA32F`) [S1, unofficial].
*   **Size Limit:** Constrained by video memory and maximum texture resolution/format supported by hardware [S1, unofficial].
*   **Access:** Retrieved via texel-fetching instructions (e.g., `texelFetch`) in the vertex shader. The instance index is used as the texture coordinate.

#### 3. Regular Vertex Buffers
*   **Layout:** Standard vertex attribute buffers. Each instance is treated as a distinct vertex or a set of vertices with associated attributes.
*   **Size Limit:** Restricted only by available video memory [S1, unofficial].
*   **Access:** Hardware loads per-instance data into registers prior to invoking the vertex program. No instance identifier is needed [S1, unofficial].

### Numbered Steps

1.  **Data Preparation:** Encode per-instance data into the chosen buffer type (Uniform, Texture, or Vertex Buffer).
2.  **Buffer Binding:** Bind the buffer to the appropriate shader stage (Uniform Block, Texture Unit, or Vertex Attribute Array).
3.  **Draw Call:** Issue a single instanced draw call (e.g., `glDrawArraysInstanced`) with the instance count equal to the number of valid instances [S1, unofficial].
4.  **Shader Execution:**
    *   *Uniform:* Shader reads from uniform arrays.
    *   *Texture:* Shader fetches texels using instance-derived coordinates.
    *   *Vertex Buffer:* Hardware automatically loads attributes into registers; shader accesses them as standard vertex attributes.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Example for Regular Vertex Buffers (most common for large datasets)
# Assume 'instance_data' is a buffer containing N instances, each with a transformation matrix (16 floats)
# and a base vertex buffer containing the unit cube (8 vertices).

def render_per_instance_vertex_buffer(instance_data, base_geometry, instance_count):
    # 1. Bind instance data as a vertex attribute
    # The hardware treats each instance's data as if it were part of the vertex stream
    bind_vertex_attribute(buffer=instance_data, attribute_index=1, stride=64) # 16 floats * 4 bytes
    
    # 2. Bind base geometry
    bind_vertex_buffer(buffer=base_geometry, attribute_index=0)
    
    # 3. Issue instanced draw call
    # The driver/hardware handles the repetition of base geometry for each instance
    # and loads the corresponding instance attribute data into registers
    draw_arrays_instanced(
        mode=TRIANGLES,
        first=0,
        count=base_geometry.vertex_count,
        instance_count=instance_count
    )

def vertex_shader():
    # Access base vertex position
    pos = vertex_attribute(0)
    
    # Access instance-specific transformation matrix
    # No gl_InstanceID lookup needed; hardware provides the correct instance's data
    mat = vertex_attribute(1) 
    
    # Compute final position
    final_pos = mat * vec4(pos, 1.0)
    return final_pos
```

### Edge Cases and Pitfalls
*   **Uniform Buffer Size:** Exceeding the 64 KB limit requires multiple draw calls or buffer subregions [S1, unofficial].
*   **Texture Encoding:** Data must be correctly encoded/decoded according to pixel format, adding overhead [S1, unofficial].
*   **Precision:** Texture-based lookups may suffer from precision loss if not using high-precision formats [S1, unofficial].

## Subtopics

### Uniform Buffers
Best for small datasets (<64KB) due to simplicity. Limited by uniform buffer size restrictions [S1, unofficial].

### Textures and Texture Buffers
Useful when data fits well into texture formats or when uniform limits are hit. Requires encoding/decoding overhead [S1, unofficial].

### Regular Vertex Buffers
Best for large datasets limited only by video memory. Hardware handles data loading, avoiding dynamic indexing penalties [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per vertex for data access (hardware registers) or $O(1)$ for uniform/texture lookups. No dynamic branching for instance ID calculation.
*   **Memory Complexity:** Proportional to the number of instances and data size per instance. Vertex buffers are limited only by video memory [S1, unofficial].
*   **Performance:** Regular vertex buffers are generally more efficient for large instance counts as they avoid the dynamic SSBO lookup penalty associated with `gl_InstanceID` [S1, S2, S4, unofficial]. Uniform buffers are fast but limited in size. Texture buffers add encoding/decoding overhead.
*   **Strengths:** No need for explicit instance ID management in shader; hardware handles data loading for vertex buffers.
*   **Limitations:** Uniform buffers have strict size limits; texture buffers require data encoding.

## Background
This method is described in the context of hardware-accelerated geometry instancing for surfel and voxel rendering [S1, unofficial]. It represents an alternative to dedicated instancing mechanisms by utilizing existing buffer types.

## Sources
[S1] Hardware-accelerated geometry instancing for surfel and voxel rendering of scanned 4d media, http://133.5.19.150/papers/2010_2013/1-1-36.pdf, fetched 2026-10-31 (unofficial)
